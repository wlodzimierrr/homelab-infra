# Homelab Web Service Operations

This runbook is the operator entrypoint for the `homelab-web` service.

## Service map

- Service ID: `homelab-web`
- Source repo: `apps/portal/frontend`
- GitOps overlays:
  - `workloads/apps/homelab-web/envs/dev`
  - `workloads/apps/homelab-web/envs/prod`
- Dev Argo application: `homelab-web-dev`
- Prod intent: Git-only overlay in the current single-cluster safety mode

## Standard checks

```bash
kubectl -n homelab-web get deploy,svc,ingress,pods
kubectl -n homelab-web rollout status deployment/homelab-web --timeout=180s
kubectl -n argocd get application homelab-web-dev
```

## Promotion and rollback

- Promotion contract: `docs/runbooks/gitops-dev-prod-promotion.md`
- Validate the GitOps repo before merging:

```bash
cd workloads
./scripts/check-environment-contract.sh
```

## Service-specific operations

- Portal auth gate and emergency bypass: `workloads/README.md`
- SSO break glass: `docs/runbooks/sso-break-glass.md`
- UI release and status join review: `docs/runbooks/release-traceability-dashboard.md`

## Monitoring and triage

- Platform health dashboard: `docs/runbooks/platform-health-dashboard.md`
- Logs workflow: `docs/runbooks/log-to-triage-workflow.md`
- Uptime indicator: `docs/runbooks/uptime-indicator-widget.md`
