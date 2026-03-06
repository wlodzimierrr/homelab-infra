# Project Registry Sync Pipeline

This runbook validates GitOps-backed project sync (`T4.7.2`).

## Scope

- Reads GitOps app definitions from `workloads/apps/*/envs/*`.
- Projects canonical rows into `project_registry` with `source=gitops_apps`.
- Updates provenance (`sourceRef`) and prunes stale GitOps-owned rows for the synced env scope.

## Trigger Sync

Admin-authenticated API trigger:

```bash
curl -sS -X POST \
  -H 'Authorization: Bearer dev-static-token' \
  'http://api.dev.homelab.local/service-registry/sync?source=gitops_apps&env=dev' | jq
```

Local script trigger:

```bash
cd apps/portal/backend
source .venv/bin/activate
python scripts/sync_project_registry.py
```

Combined catalog trigger:

```bash
cd apps/portal/backend
source .venv/bin/activate
python scripts/sync_catalog_registries.py
```

## Expected Response Fields

- `correlationId`
- `source=gitops_apps`
- `env`
- `namespaces`
- `discovered`
- `upserted`
- `inserted`
- `updated`
- `deleted`
- `sourceFailures`
- `generatedAt`
- `durationMs`

## Validation

1. Run the sync twice and verify the second run is idempotent (`inserted=0` unless manifests changed).
2. `GET /projects?env=dev` returns only GitOps-backed rows.
3. Remove or rename a GitOps app env directory and rerun sync; the stale row is removed or updated deterministically.

## Failure Modes

- Missing workloads repo checkout or bad `GITOPS_WORKLOADS_REPO_PATH` yields `sourceFailures`.
- Invalid or incomplete app metadata falls back to defaults where safe; unresolved metadata issues are reported in `sourceFailures`.

## Scheduled Sync

The in-cluster scheduler runs the combined catalog sync every 10 minutes:

- manifest: `workloads/apps/homelab-api/base/catalog-sync-cronjob.yaml`
- output: JSON summary including both `gitops_apps` and `cluster_services`
- failure behavior: CronJob exits non-zero when either source reports `sourceFailures`
