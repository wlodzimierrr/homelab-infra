# Catalog Sync Schedule and Freshness SLO

This runbook validates `T4.7.7` for scheduled catalog sync and pre-stale freshness warnings.

## Scope

- Periodic project sync from GitOps app definitions
- Periodic service sync from live cluster resources
- Freshness warning before catalog data becomes stale
- Cluster troubleshooting for failed scheduled runs

## Scheduler

The in-cluster scheduler is:

- manifest: `workloads/apps/homelab-api/base/catalog-sync-cronjob.yaml`
- resource: `CronJob/homelab-api-catalog-sync`
- schedule: every 10 minutes (`*/10 * * * *`)
- command: `python scripts/sync_catalog_registries.py`

The job runs both:

- `gitops_apps` -> `project_registry`
- `cluster_services` -> `service_registry`

## Freshness SLO

Diagnostics expose:

- `freshness.warningAfterMinutes`
- `freshness.staleAfterMinutes`
- `freshness.isWarning`
- `freshness.isStale`
- `freshness.state` = `fresh | warning | stale | empty`

Expected default behavior:

- `warningAfterMinutes`: 66% of `REGISTRY_STALE_AFTER_MINUTES`
- `staleAfterMinutes`: `30`
- `warning` should appear before `stale`

Override with:

- `REGISTRY_WARN_AFTER_MINUTES`
- `REGISTRY_STALE_AFTER_MINUTES`

## Cluster Validation

1. Verify the scheduler exists:

```bash
kubectl -n homelab-api get cronjob homelab-api-catalog-sync
```

2. Force an immediate run if needed:

```bash
kubectl -n homelab-api create job --from=cronjob/homelab-api-catalog-sync catalog-sync-manual-$(date +%s)
```

3. Inspect recent runs:

```bash
kubectl -n homelab-api get jobs --sort-by=.metadata.creationTimestamp | tail
kubectl -n homelab-api logs job/<latest-catalog-sync-job>
```

4. Confirm diagnostics show healthy/fresh state:

```bash
curl -sS -H 'Authorization: Bearer dev-static-token' \
  'http://api.dev.homelab.local/projects/diagnostics?env=dev' | jq
curl -sS -H 'Authorization: Bearer dev-static-token' \
  'http://api.dev.homelab.local/service-registry/diagnostics?env=dev' | jq
```

## Failure Triage

- CronJob `Last Schedule Time` missing: scheduler not applied or suspended
- Job pod exits non-zero: inspect JSON output from `scripts/sync_catalog_registries.py`
- `hasFailures=true`: one or both source summaries include `sourceFailures`
- `freshness.state=warning`: sync is aging and should refresh before SLO breach
- `freshness.state=stale`: scheduled sync is blocked or not running

Expected log shape for a healthy run:

```text
catalog_sync_source_result env=dev source=gitops_apps correlation_id=<uuid> discovered=2 upserted=2 inserted=0 updated=2 deleted=0 failures=0 duration_ms=<n>
catalog_sync_source_result env=dev source=cluster_services correlation_id=<uuid> discovered=4 upserted=3 inserted=0 updated=3 deleted=0 failures=0 duration_ms=<n>
catalog_sync_run_result env=dev has_failures=False exit_code=0 gitops_failures=0 cluster_failures=0 gitops_correlation_id=<uuid> cluster_correlation_id=<uuid>
```

Expected failure signals:

- `catalog_sync_run_result ... has_failures=True exit_code=1`
- `catalog_sync_run_error ...` when the job fails before source summaries are generated
- source-specific backend logs such as `service_registry_sync_source_error ...`

Recommended checks for alerting/triage:

```bash
kubectl -n homelab-api logs job/<latest-catalog-sync-job> | rg 'catalog_sync_(source_result|run_result|run_error)|service_registry_sync_source_error'
kubectl -n homelab-api get jobs --sort-by=.metadata.creationTimestamp | tail -n 3
```

## Evidence Files

- `workloads/apps/homelab-api/base/catalog-sync-cronjob.yaml`
- `apps/portal/backend/scripts/sync_catalog_registries.py`
- `apps/portal/backend/app/main.py`
- `apps/portal/backend/tests/test_api.py`
