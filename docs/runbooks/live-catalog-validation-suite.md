# Live Catalog Validation Suite

This runbook validates `T4.7.8` for end-to-end live projects/services/monitoring flow.

## Scope

The suite verifies:

- GitOps apps -> `/projects`
- project freshness -> `/projects/diagnostics`
- live cluster services -> `/services`
- service freshness -> `/service-registry/diagnostics`
- project/service joins -> `/releases`
- monitoring summary data -> `/services/:serviceId/metrics/summary`
- live alerts feed -> `/alerts/active`

## Scripted Validation

From `apps/portal/backend`:

```bash
source .venv/bin/activate
python scripts/live_catalog_validation.py \
  --api-base-url http://api.dev.homelab.local \
  --auth-token dev-static-token \
  --env dev \
  --report-file /tmp/live-catalog-validation-dev.json
```

The script fails if:

- legacy seeded/default project rows are detected
- `/projects/diagnostics` or `/service-registry/diagnostics` report `state=stale`
- `/projects/diagnostics` or `/service-registry/diagnostics` report `state=empty`
- `/releases` contains rows not present in canonical `/projects` or `/services`
- metrics provider is not healthy
- alerts provider is not healthy

The script emits a JSON report with:

- `generatedAt`
- `env`
- `serviceId`
- `summary.projects`
- `summary.projectDiagnostics`
- `summary.services`
- `summary.serviceDiagnostics`
- `summary.releases`
- `summary.metrics`
- `summary.alerts`
- `warnings`

## Before/After Evidence Capture

Capture a baseline before a sync, rollout, or metadata fix:

```bash
source .venv/bin/activate
python scripts/live_catalog_validation.py \
  --api-base-url http://api.dev.homelab.local \
  --auth-token dev-static-token \
  --env dev \
  --report-file /tmp/live-catalog-before.json
```

Apply the remediation or run scheduled/manual sync, then capture the after report:

```bash
source .venv/bin/activate
python scripts/live_catalog_validation.py \
  --api-base-url http://api.dev.homelab.local \
  --auth-token dev-static-token \
  --env dev \
  --report-file /tmp/live-catalog-after.json
```

Record:

- before report path
- after report path
- warnings present/cleared
- any seeded legacy rows removed
- freshness state transitions (`warning/stale -> fresh`)

## Failure Triage

- Legacy project rows: inspect `/projects` source-of-truth and remove seeded/manual leftovers
- Stale diagnostics: run catalog sync or inspect `CronJob/homelab-api-catalog-sync`
- Release join failures: inspect `/catalog/reconciliation` and `/releases`
- Metrics provider unhealthy: inspect `/monitoring/providers/diagnostics` and Prometheus connectivity
- Alerts provider unhealthy: inspect `/monitoring/providers/diagnostics` and Alertmanager connectivity

## Related Files

- `apps/portal/backend/scripts/live_catalog_validation.py`
- `apps/portal/backend/tests/test_live_catalog_validation.py`
- `docs/runbooks/project-source-cutover-validation.md`
- `docs/runbooks/live-data-validation-suite.md`
