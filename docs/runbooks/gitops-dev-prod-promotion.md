# GitOps Dev/Prod Promotion Contract

This runbook defines the supported GitOps promotion path for the current homelab workload repo: `dev -> prod`.

## Source of truth

- Dev root app: `workloads/bootstrap/root-app-dev.yaml`
- Prod root app: `workloads/bootstrap/root-app-prod.yaml`
- Dev overlay entrypoints:
  - `workloads/environments/dev`
  - `workloads/platform/envs/dev`
  - `workloads/apps/homelab-api/envs/dev`
  - `workloads/apps/homelab-web/envs/dev`
- Prod overlay entrypoints:
  - `workloads/environments/prod`
  - `workloads/platform/envs/prod`
  - `workloads/apps/homelab-api/envs/prod`
  - `workloads/apps/homelab-web/envs/prod`

Environment differences must stay in overlay patches and overlay-only resources. Do not fork base manifests per environment.

## Current cluster safety mode

- `workloads/environments/prod/workloads/kustomization.yaml` is intentionally empty.
- `workloads/environments/prod/workloads-app.yaml` keeps `allowEmpty: true`.
- Result: prod intent is tracked in Git, but this single cluster does not reconcile separate prod workload apps until dedicated prod namespaces or a prod cluster exist.

## Supported mutation paths

Normal image movement is limited to these workflows:

1. Dev deploys:
   - Workflow: `apps/portal/.github/workflows/portal-images.yml`
   - Effect: opens a PR that updates only dev image patch files.
2. Prod promotions or rollbacks:
   - Workflow: `apps/portal/.github/workflows/gated-promotion.yml`
   - Trigger: `workflow_dispatch`
   - Target: `prod`
   - Modes:
     - `promote`: copy deployed tags from dev overlays into prod overlays
     - `rollback`: write explicit rollback tags into prod overlays after approval

For `homelab-api` and `homelab-web`, a normal prod promotion PR may change only:

- `workloads/apps/homelab-api/envs/prod/patch-deployment.yaml`
- `workloads/apps/homelab-api/envs/prod/patch-migration-job.yaml`
- `workloads/apps/homelab-api/envs/prod/patch-catalog-sync-cronjob.yaml`
- `workloads/apps/homelab-web/envs/prod/patch-deployment.yaml`

If a PR needs to touch more than those files, treat it as an explicit environment-change review, not a routine promotion.

## Validation

Run these from `workloads/` before merging structure changes:

```bash
./scripts/check-rbac-guardrails.sh
./scripts/check-secrets-guardrails.sh
./scripts/check-environment-contract.sh
```

The GitOps repo also runs `.github/workflows/validate-gitops-environment-contract.yml` on pushes and pull requests that modify manifests or guardrail scripts.

## Rollback

Use one of these paths:

1. Preferred: re-run `apps/portal/.github/workflows/gated-promotion.yml` with `action_mode=rollback` and explicit known-good tags.
2. Fallback: revert the merged prod promotion PR in the GitOps repo.

Both paths preserve Git as the deployment audit trail and keep Argo CD as the reconciler.
