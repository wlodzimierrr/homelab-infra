# Service Health Timeline Endpoint Runbook

This runbook validates backend ticket T4.4.4.

## 1. Scope

Endpoint:

- `GET /api/services/:serviceId/health/timeline?range=24h&step=5m`

Returns compact status segments:

- `{ start, end, status, reason? }`

Status values:

- `healthy`, `degraded`, `down`, `unknown`

## 2. Range and step validation

Supported windows:

- `24h`, `7d`

Step bounds:

- minimum `5m`
- maximum `1h`
- request is rejected when step would produce too many points for selected window

## 3. Status mapping inputs

Mapping uses Prometheus signals:

- availability (`up`)
- error rate (`http_requests_total` 5xx ratio)
- readiness (deployment available/spec replicas)

Thresholds are configurable via env vars:

- `TIMELINE_DEGRADED_AVAILABILITY_MAX`
- `TIMELINE_DOWN_AVAILABILITY_MAX`
- `TIMELINE_DEGRADED_ERROR_RATE_MIN_PCT`
- `TIMELINE_DOWN_ERROR_RATE_MIN_PCT`
- `TIMELINE_DEGRADED_READINESS_MAX`
- `TIMELINE_DOWN_READINESS_MAX`

## 4. Validation steps

1. Call endpoint with `range=24h&step=5m`; verify HTTP 200 and segment array shape.
2. Call endpoint with `range=7d&step=1h`; verify bounded response.
3. Call endpoint with invalid step (`1m`); verify HTTP 422.
4. Force missing Prometheus series and verify endpoint still returns `unknown` segments instead of crashing.
5. Simulate Prometheus non-200 and verify stable HTTP 502 with correlation ID in detail.

## 5. Evidence files

- `apps/portal/backend/app/main.py`
- `apps/portal/backend/app/health_timeline.py`
- `apps/portal/backend/tests/test_health_timeline.py`
- `apps/portal/backend/tests/test_api.py`
