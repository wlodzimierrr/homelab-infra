# Quarterly Secret Rotation Runbook

This runbook covers T3.3.2 for quarterly rotation of:

1. API Postgres credentials (SOPS-managed)
2. oauth2-proxy credentials/cookie secret
3. GHCR pull secrets

Target: no prolonged outage (rollout completion <= 5 minutes per rotated workload).

## 1. Pre-checks

```bash
cd workloads
./scripts/check-secrets-guardrails.sh
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

Create new GitHub client secret and cookie secret, then update Kubernetes Secret:

```bash
kubectl -n homelab-web create secret generic oauth2-proxy-secret \
  --from-literal=OAUTH2_PROXY_CLIENT_ID='<PORTAL_GH_CLIENT_ID>' \
  --from-literal=OAUTH2_PROXY_CLIENT_SECRET='<NEW_PORTAL_GH_CLIENT_SECRET>' \
  --from-literal=OAUTH2_PROXY_COOKIE_SECRET="$(openssl rand -base64 32)" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n homelab-web rollout restart deployment/oauth2-proxy
```

Validate rollout SLO:

```bash
./workloads/scripts/verify-rotation-slo.sh homelab-web oauth2-proxy 300
```

## 4. Rotate GHCR pull secrets

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

## 5. Post-rotation validation

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

## 6. Rollback

If rotation fails:

1. Revert SOPS secret commit and push.
2. Re-apply previous known-good runtime secret for oauth2-proxy/GHCR.
3. Restart affected deployment.
4. Confirm service restored, then perform root-cause analysis.
