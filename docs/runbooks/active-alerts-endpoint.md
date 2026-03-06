# Active Alerts Endpoint Runbook

This runbook validates T4.4.10 for `/api/alerts/active`.

## 1. Scope

Backend must expose active alerts mapped to portal severities:

- `GET /api/alerts/active?env=...&serviceId=...`

Expected response shape:

- `{ alerts: [...], providerStatus }`
- alert row: `{ id, severity, title, description?, startsAt, labels, serviceId?, env? }`

Behavior:

- severities normalized to `warning|critical`
- supports filtering by `env` and `serviceId`
- upstream failures degrade to HTTP `200` with `alerts=[]` plus degraded `providerStatus`
- compatibility route remains available:
  - `GET /api/monitoring/incidents`

## 2. Configuration

- `ALERTMANAGER_BASE_URL`
- `ALERT_SEVERITY_CRITICAL_VALUES`
- `ALERT_SEVERITY_WARNING_VALUES`

## 3. Validation steps

1. Call `GET /api/alerts/active` with auth and verify envelope response.
2. Confirm each alert item includes `id`, `severity`, `title`, `startsAt`, `labels`.
3. Confirm `providerStatus.provider=alertmanager`.
4. Verify only `warning|critical` severities are returned.
5. Call with `env=dev` and `serviceId=homelab-api`; verify filtered rows only.
6. Simulate Alertmanager upstream error; verify endpoint returns `200`, `alerts=[]`, and degraded `providerStatus` with `correlationId`.
7. Call `GET /api/monitoring/incidents`; verify `{"incidents":[...]}` compatibility shape.

## 4. Evidence files

- `apps/portal/backend/app/main.py`
- `apps/portal/backend/app/alerts_feed.py`
- `apps/portal/backend/tests/test_api.py`
- `apps/portal/backend/tests/test_alerts_feed.py`
