# kube-prometheus-stack Runbook (Homelab)

This runbook covers deployment and validation of `kube-prometheus-stack` for T4.1.1 with low retention and bounded storage for homelab capacity.

## 1. Scope and paths

Argo CD applications:

1. `workloads/environments/dev/workloads/monitoring-app.yaml`
2. No separate `monitoring-prod` application in this cluster (single shared monitoring stack).

Chart source:

1. Helm repo: `https://prometheus-community.github.io/helm-charts`
2. Chart: `kube-prometheus-stack`
3. Pinned chart version: `69.5.2`

## 2. Homelab resource profile

Dev:

1. Prometheus retention: `24h`
2. Prometheus retention size cap: `3GiB`
3. Prometheus PVC request: `5Gi`
4. Grafana PVC request: `2Gi`

Shared settings:

1. Alertmanager disabled (`alertmanager.enabled=false`) for reduced baseline usage.
2. `kube-state-metrics` and `prometheus-node-exporter` enabled for cluster/node/pod visibility.
3. Requests/limits applied to Prometheus, Grafana, operator, and exporters.
4. AppProject `homelab-monitoring` must allow both `monitoring` and `kube-system` destinations because the chart creates control-plane scrape Service objects in `kube-system`.
5. Current strategy is one shared monitoring release in namespace `monitoring`; do not deploy the same release into prod app-of-apps without separate namespace/release naming.

## 3. Deploy and sync

```bash
kubectl apply -f workloads/bootstrap/project-homelab.yaml
kubectl apply -k workloads/environments/dev/workloads
```

Verify Argo CD sync:

```bash
argocd app get monitoring-dev --grpc-web
```

## 4. Validate metrics visibility

Confirm core monitoring pods:

```bash
kubectl -n monitoring get pods
```

Port-forward Grafana and open dashboards:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Expected:

1. Dashboard `Kubernetes / Compute Resources / Cluster` shows cluster CPU/memory.
2. Dashboard `Kubernetes / Compute Resources / Node (Pods)` shows node-level metrics.
3. Dashboard `Kubernetes / Compute Resources / Pod` shows pod-level CPU/memory.

## 5. Validate retention and storage usage

Prometheus runtime config:

```bash
kubectl -n monitoring get prometheus -o jsonpath='{range .items[*]}{.metadata.name}{" retention="}{.spec.retention}{" retentionSize="}{.spec.retentionSize}{"\n"}{end}'
```

PVC allocation and current usage:

```bash
kubectl -n monitoring get pvc
kubectl -n monitoring exec -it sts/prometheus-kube-prometheus-stack-prometheus -- df -h /prometheus
```

## 6. Capacity notes

Use this simple guardrail for Prometheus disk planning:

1. Keep `retentionSize` under ~70% of requested PVC.
2. Keep at least 1Gi free for WAL/head compaction spikes.
3. If usage grows, increase PVC first, then tune retention window.
4. For future env split, deploy distinct releases (for example `kube-prometheus-stack-dev` and `kube-prometheus-stack-prod`) into separate namespaces.

## 7. Rollback

If monitoring rollout causes pressure:

1. Disable prod app auto-sync temporarily in Argo CD.
2. Revert monitoring app manifest commit.
3. Resync app and verify Prometheus/Grafana pods return to previous revision.
