# Service Metrics Summary Endpoint Runbook

This runbook validates backend endpoint `T4.4.2`.

## 1. Scope

Endpoint:

- `GET /api/services/:serviceId/metrics/summary?range={1h|24h|7d}`

Returns:

- `uptimePct`, `p95LatencyMs`, `errorRatePct`, `restartCount`
- `windowStart`, `windowEnd`, `generatedAt`
- `noData` flags per metric
- `providerStatus` for Prometheus query readiness

Compatibility alias also exists:

- `GET /api/services/:serviceId/metrics-summary?range=...`

Current default behavior:

- `uptimePct` falls back to deployment availability over the selected window using kube-state-metrics.
- `restartCount` uses container restart counters from Kubernetes metrics.
- `p95LatencyMs` and `errorRatePct` require service-level HTTP instrumentation. If the app does not emit matching request metrics, these fields will remain `noData=true` even when Prometheus itself is healthy.

## 2. Validation steps

1. Call endpoint with a valid bearer token and `range=24h`.
2. Verify HTTP 200 and response shape includes all fields above.
3. Repeat with `range=1h` and `range=7d`.
4. Call endpoint with invalid range (`range=2h`) and verify HTTP 422.
5. Force empty Prometheus series (or query a non-existent service) and verify per-metric `noData` flags instead of full-request failure.
6. Simulate Prometheus non-200 and verify backend returns stable HTTP 502 with structured error payload:
   - `detail.message`
   - `detail.correlationId`
   - `detail.providerStatus.provider=prometheus`
   - `detail.providerStatus.status=http_error|auth_error`

## 3. Prometheus env configuration

- `PROMETHEUS_BASE_URL` (default: `http://prometheus-operated.monitoring.svc.cluster.local:9090`)
- `PROMETHEUS_TIMEOUT_SECONDS` (default: `8`)

## 4. Evidence files

- `apps/portal/backend/app/main.py`
- `apps/portal/backend/tests/test_api.py`
