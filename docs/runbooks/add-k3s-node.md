# Add k3s Node Runbook

This runbook adds a new node to the existing homelab cluster using current Day-0 automation.

## 1. Scope

Supported by current repo automation:

1. Add a new `k3s server` (control-plane/etcd member) via Ansible inventory + playbooks.

Not currently modeled:

1. Worker-only (`k3s agent`) join flow. The repo has `k3s_servers` + `k3s_nodes`, but no agent group/role.

## 2. Prerequisites

1. New node OS is installed and reachable over LAN SSH as `sysadmin` with sudo.
2. Node is on the same LAN subnet pattern used here (`192.168.50.0/24`).
3. You have Ansible Vault password for `ansible/inventory/group_vars/k3s_servers/vault.yml` (`vault_k3s_token`).
4. Existing cluster is healthy before change:

```bash
kubectl get nodes -o wide
ansible all -m ping
```

## 3. Pick node values

Choose new values that do not conflict:

1. Inventory hostname (example: `node-4`)
2. LAN IP (example: `192.168.50.30`)
3. WireGuard IP (example: `10.100.0.14/32`)

## 4. Update inventory and host vars

### 4.1 Add host to both inventory groups

Edit `ansible/inventory/hosts.ini` and add the node in both sections:

1. `[k3s_nodes]` uses LAN `ansible_host`
2. `[k3s_servers]` uses WireGuard `ansible_host`

Example:

```ini
[k3s_nodes]
node-4 ansible_host=192.168.50.30 ansible_user=sysadmin ansible_become=true

[k3s_servers]
node-4 ansible_host=10.100.0.14 ansible_user=sysadmin ansible_become=true
```

### 4.2 Create host vars file

Create `ansible/inventory/host_vars/node-4.yml`:

```yaml
wg_client_ip: "10.100.0.14/32"
k3s_node_ip: 192.168.50.30
```

Notes:

1. Do not add `wg_extra_allowed_ips` unless this node should advertise extra routes (today only `node-1` advertises Traefik VIP route).
2. Ensure `k3s_node_ip` and `wg_client_ip` are unique.

## 5. Validate connectivity to the new host

Run from `ansible/`:

```bash
cd ansible
ansible node-4 -m ping
ansible k3s_nodes -m ping
```

## 6. Apply WireGuard update

Run WireGuard playbook without `--limit` so peer key map includes all nodes:

```bash
ansible-playbook playbooks/wireguard.yml
```

Quick checks:

```bash
ansible node-4 -m shell -a 'ip a show wg0'
ansible gateway -m shell -a 'wg show'
```

## 7. Join node to k3s HA cluster

Join only the new server:

```bash
ansible-playbook playbooks/k3s_ha.yml --limit node-4
```

Why this works:

1. Role logic treats non-first `k3s_servers` as joiners.
2. Existing nodes are already installed and skipped by binary check.

## 8. Post-join validation

Run from any kubeconfig context that targets the cluster:

```bash
kubectl get nodes -o wide
kubectl get node node-4
kubectl -n kube-system get pods -o wide
```

Expected:

1. `node-4` is `Ready`.
2. Core system pods remain healthy.
3. Existing ingress/app traffic is unaffected.

Optional scheduling check:

```bash
kubectl get pods -A -o wide | rg node-4
```

## 9. Troubleshooting

1. `ansible node-4 -m ping` fails:
   - Verify SSH key/user/sudo on target host.
   - Verify LAN IP and routing.
2. WireGuard up but k3s join fails:
   - Check `k3s_token` from vault.
   - Verify API server reachable from node-4:
     `curl -k https://<first-server-k3s-node-ip>:6443/healthz`
3. Node not `Ready`:
   - Check `sudo systemctl status k3s` on node-4.
   - Check CNI/flannel interface matches `k3s_flannel_iface` in group vars.

## 10. Rollback (failed add)

If node add must be reverted:

1. On node-4:
   - `sudo /usr/local/bin/k3s-uninstall.sh` (if installed)
   - `sudo systemctl stop wg-quick@wg0`
2. Remove `node-4` entries from:
   - `ansible/inventory/hosts.ini`
   - `ansible/inventory/host_vars/node-4.yml`
3. Re-run:

```bash
cd ansible
ansible-playbook playbooks/wireguard.yml
kubectl get nodes -o wide
```

## 11. Operational note

Per ADR-0005, node lifecycle is Day-0 ownership (Terraform/Ansible), not a portal Day-2 workflow.
