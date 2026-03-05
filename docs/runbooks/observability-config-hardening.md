# Observability Config Hardening Runbook

This runbook validates T4.4.12.

## 1. Scope

Verify monitoring endpoints are bounded and cache-backed:

- `GET /api/services/:serviceId/metrics/summary`
- `GET /api/services/:serviceId/health/timeline`
- `GET /api/services/:serviceId/logs/quickview`
- `GET /api/alerts/active`

## 2. Configuration checks

1. Verify env knobs are set (or defaults used):
   - ranges: `OBS_*_ALLOWED_RANGES`
   - limits: `OBS_TIMELINE_*`, `OBS_LOGS_MAX_LINES`, `OBS_ALERTS_MAX_ROWS`
   - cache TTLs: `OBS_*_CACHE_TTL_SECONDS`
2. Verify query templates in:
   - `docs/monitoring/query-templates.md`

## 3. Functional checks

1. Call metrics endpoint repeatedly with same service/range and verify stable output and no provider overload.
2. Call timeline endpoint with out-of-bound step and verify 422 response.
3. Call logs quick-view with high `limit` and verify response limit is capped by config.
4. Call alerts endpoint with high `limit` and verify rows are capped by config.

## 4. Repeated-refresh smoke test

Run:

```bash
cd apps/portal/backend
python3 scripts/monitoring_refresh_smoke.py --base-url http://127.0.0.1:8000 --service-id homelab-api --iterations 20
```

Expected:

- all requests return HTTP 200
- summary prints p50/p95/max latency
- failure count is `0`

## 5. Evidence files

- `apps/portal/backend/app/observability_config.py`
- `apps/portal/backend/app/observability_cache.py`
- `apps/portal/backend/app/main.py`
- `docs/monitoring/query-templates.md`
- `apps/portal/backend/scripts/monitoring_refresh_smoke.py`
