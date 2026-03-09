# Homelab API Service Operations

This runbook is the operator entrypoint for the `homelab-api` service.

## Service map

- Service ID: `homelab-api`
- Source repo: `apps/portal/backend`
- GitOps overlays:
  - `workloads/apps/homelab-api/envs/dev`
  - `workloads/apps/homelab-api/envs/prod`
- Dev Argo application: `homelab-api-dev`
- Prod intent: Git-only overlay in the current single-cluster safety mode

## Standard checks

```bash
kubectl -n homelab-api get deploy,svc,ingress,pods
kubectl -n homelab-api rollout status deployment/homelab-api --timeout=180s
kubectl -n argocd get application homelab-api-dev
```

## Promotion and rollback

- Promotion contract: `docs/runbooks/gitops-dev-prod-promotion.md`
- Validate the GitOps repo before merging:

```bash
cd workloads
./scripts/check-environment-contract.sh
```

## Service-specific operations

- Secrets and Postgres bootstrap: `workloads/README.md`
- Catalog sync validation: `docs/runbooks/live-catalog-validation-suite.md`
- Project registry sync: `docs/runbooks/project-registry-sync-pipeline.md`
- Service registry sync: `docs/runbooks/service-registry-sync-pipeline.md`

## Monitoring and triage

- Metrics summary: `docs/runbooks/service-metrics-summary-endpoint.md`
- Health timeline: `docs/runbooks/service-health-timeline-endpoint.md`
- Logs workflow: `docs/runbooks/log-to-triage-workflow.md`
