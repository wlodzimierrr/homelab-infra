# Quarterly Secret Rotation Runbook

This runbook covers T3.3.2 for quarterly rotation of:

1. API Postgres credentials (SOPS-managed)
2. oauth2-proxy credentials/cookie secret
3. Portal GitHub Actions dispatch token
4. Portal Git write token
5. GHCR pull secrets

Target: no prolonged outage (rollout completion <= 5 minutes per rotated workload).

## 1. Pre-checks

```bash
cd workloads
./scripts/check-secrets-guardrails.sh
./scripts/render-kustomize.sh apps/homelab-api/envs/dev >/dev/null
./scripts/render-kustomize.sh apps/homelab-web/envs/dev >/dev/null
kubectl -n homelab-api get deploy homelab-api
kubectl -n homelab-web get deploy homelab-web oauth2-proxy
```

## 2. Rotate Postgres credentials (dev/prod)

Re-encrypt secret values:

```bash
cd workloads
./scripts/bootstrap-sops-postgres-secret.sh dev
./scripts/bootstrap-sops-postgres-secret.sh prod
```

Commit/push encrypted updates:

```bash
git add apps/homelab-api/envs/dev/postgres-secret.enc.yaml \
  apps/homelab-api/envs/prod/postgres-secret.enc.yaml
git commit -m "chore(secrets): quarterly rotate postgres credentials"
git push origin main
```

Validate rollout SLO:

```bash
./scripts/verify-rotation-slo.sh homelab-api homelab-api 300
```

## 3. Rotate oauth2-proxy credentials

Create a new GitHub client secret and re-encrypt the Git-managed secret:

```bash
cd workloads
./scripts/bootstrap-sops-oauth2-proxy-secret.sh dev
git add apps/homelab-web/envs/dev/oauth2-proxy-secret.enc.yaml
git commit -m "chore(secrets): quarterly rotate oauth2-proxy credentials"
git push origin main
```

Validate rollout SLO:

```bash
./scripts/verify-rotation-slo.sh homelab-web oauth2-proxy 300
```

## 4. Rotate portal GitHub Actions dispatch token

Generate a replacement token with only the scopes needed to dispatch the portal GitHub workflow, re-encrypt it into Git, and let Argo roll out the API Deployment:

```bash
cd workloads
./scripts/bootstrap-sops-github-actions-token.sh dev
git add apps/homelab-api/envs/dev/github-actions-secret.enc.yaml
git commit -m "chore(secrets): rotate portal GitHub Actions token"
git push origin main
```

Validate rollout SLO:

```bash
./scripts/verify-rotation-slo.sh homelab-api homelab-api 300
kubectl -n homelab-api get secret homelab-api-github-actions
```

## 5. Rotate portal Git write token

Create a fine-grained token scoped only to `wlodzimierrr/homelab-workloads` with:

- repository access: single repo
- permissions:
  - `Contents`: Read and write
  - `Pull requests`: Read and write
  - `Metadata`: Read-only

Re-encrypt it into Git and verify read access before relying on it from the deployed API:

```bash
cd workloads
./scripts/bootstrap-sops-git-github-token.sh --prompt dev
git add \
  apps/homelab-api/envs/dev/git-github-token-secret.enc.yaml \
  apps/homelab-api/envs/dev/git-github-token-secret-generator.yaml \
  apps/homelab-api/envs/dev/kustomization.yaml
git commit -m "chore(secrets): rotate portal git write token"
git push origin main

cd ../apps/portal/backend
GIT_GITHUB_TOKEN="$NEW_GIT_GITHUB_TOKEN" ./.venv/bin/python scripts/verify_git_github_token.py
```

Validate rollout SLO:

```bash
cd ../../workloads
./scripts/verify-rotation-slo.sh homelab-api homelab-api 300
kubectl -n homelab-api get secret homelab-api-git-github
```

If the workstation age key was rotated in the same quarter, re-apply the runtime age key used by the portal secret-edit endpoint:

```bash
cd workloads
./scripts/bootstrap-runtime-sops-age-key.sh dev
kubectl -n homelab-api rollout restart deploy/homelab-api
kubectl -n homelab-api rollout status deploy/homelab-api --timeout=300s
```

## 6. Rotate GHCR pull secrets

Issue new read-only package token and update both namespaces:

```bash
kubectl -n homelab-api create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=wlodzimierrr \
  --docker-password="$NEW_CR_PAT" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n homelab-web create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=wlodzimierrr \
  --docker-password="$NEW_CR_PAT" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Restart workloads that consume the pull secret only if required by image pull failures.

## 7. Post-rotation validation

```bash
argocd app get homelab-api-dev --grpc-web
argocd app get homelab-web-dev --grpc-web
kubectl -n homelab-api get pods
kubectl -n homelab-web get pods
```

Record:

1. Start/end timestamps for each rotated workload
2. `verify-rotation-slo.sh` output
3. Any incident, rollback, or manual intervention

## 8. Rollback

If rotation fails:

1. Revert SOPS secret commit and push.
2. Re-apply previous known-good GHCR secret if pull credentials were part of the change.
3. Restart affected deployment.
4. Confirm service restored, then perform root-cause analysis.
