# SOPS Secrets Runbook

This runbook implements T3.3.1 using `SOPS + age` for encryption and `KSOPS` for Argo CD/Kustomize decryption.

## 1. Install tooling

Install `sops`, `age`, `ksops`, and `kustomize` on your operator workstation.

## 2. Create an age keypair

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
grep '^# public key:' ~/.config/sops/age/keys.txt
```

Save the printed public key. It is used in `.sops.yaml`.

## 3. Configure repository creation rules

Create/update `workloads/.sops.yaml`:

```yaml
creation_rules:
  - path_regex: apps/.*/envs/.*/.*secret.*\.enc\.ya?ml$
    encrypted_regex: "^(data|stringData)$"
    age:
      - age1REPLACE_WITH_YOUR_PUBLIC_KEY
```

## 4. Publish the age key to Argo CD

Create/update the repo-server age key secret:

```bash
kubectl -n argocd create secret generic argocd-sops-age \
  --from-file=keys.txt="$HOME/.config/sops/age/keys.txt" \
  --dry-run=client -o yaml | kubectl apply -f -
```

## 5. Refresh Argo CD repo-server support

Apply the Argo CD role so repo-server receives `ksops`, the age key mount, and plugin-aware Kustomize build flags:

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/argocd.yml
```

## 6. Encrypt secret manifests

Bootstrap the live secret manifests:

```bash
cd workloads
./scripts/bootstrap-sops-postgres-secret.sh dev
./scripts/bootstrap-sops-postgres-secret.sh prod
./scripts/bootstrap-sops-oauth2-proxy-secret.sh dev
./scripts/bootstrap-sops-github-actions-token.sh --from-cluster dev
./scripts/bootstrap-sops-git-github-token.sh --prompt dev
```

Each secret overlay is wired through a `*-secret-generator.yaml` KSOPS generator. Do not add `*.enc.yaml` files directly to `resources:`.

## 6a. Bootstrap runtime age key for portal secret edits

`POST /services/{service_id}/config/set-secret` decrypts and re-encrypts SOPS manifests inside the `homelab-api` runtime. That path needs the age private key mounted into the backend pod, but the key itself must not be committed to Git.

Bootstrap the runtime key from the operator workstation:

```bash
cd workloads
./scripts/bootstrap-runtime-sops-age-key.sh dev
kubectl -n homelab-api rollout restart deploy/homelab-api
kubectl -n homelab-api rollout status deploy/homelab-api --timeout=300s
```

This creates the namespace Secret `homelab-api-sops-age` and mounts it at `/var/run/secrets/sops-age/keys.txt` via `SOPS_AGE_KEY_FILE`. Treat this as runtime-only operator state and re-apply it whenever the workstation age key rotates.

## 7. Validate guardrails and local render

```bash
cd workloads
./scripts/check-secrets-guardrails.sh
./scripts/render-kustomize.sh apps/homelab-api/envs/dev >/dev/null
./scripts/render-kustomize.sh apps/homelab-api/envs/prod >/dev/null
./scripts/render-kustomize.sh apps/homelab-web/envs/dev >/dev/null
```

This fails if plaintext `kind: Secret` manifests are committed without a `sops:` block.

## 8. Validate the live delivery path

If you have `sync` permission, you can trigger an Argo sync explicitly. Otherwise wait for auto-sync and use read-only checks:

```bash
argocd app get homelab-api-dev --grpc-web
argocd app get homelab-web-dev --grpc-web
kubectl -n homelab-api rollout status deploy/homelab-api --timeout=300s
kubectl -n homelab-web rollout status deploy/oauth2-proxy --timeout=300s
kubectl -n homelab-api get secret homelab-api-postgres
kubectl -n homelab-api get secret homelab-api-github-actions
kubectl -n homelab-api get secret homelab-api-git-github
kubectl -n homelab-web get secret oauth2-proxy-secret
```

## 9. Acceptance evidence for T3.3.1

Capture and store:

1. `git grep -n "^kind: Secret"` output showing only SOPS-managed files.
2. Guardrail script pass output.
3. Evidence that `argocd-repo-server` has `ksops` support and the age key mount.
4. Argo/Kubernetes output confirming `homelab-api` and `oauth2-proxy` start with decrypted secret values from Git-managed encrypted manifests.
5. Argo/Kubernetes output confirming the `homelab-api` Deployment reads `PORTAL_GITHUB_ACTIONS_TOKEN` from `homelab-api-github-actions`.
6. Dry-run output from `apps/portal/backend/scripts/verify_git_github_token.py` confirming `GIT_GITHUB_TOKEN` can read `wlodzimierrr/homelab-workloads`.
7. Runtime bootstrap output confirming `homelab-api-sops-age` exists before using the portal secret-edit endpoint.
