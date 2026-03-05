# Logs Quick-View Endpoint Runbook

This runbook validates backend ticket T4.4.8.

## 1. Scope

Endpoint:

- `GET /api/services/:serviceId/logs/quickview?preset=errors&range=1h`

Features:

- safe preset-only queries (`errors`, `restarts`, `warnings`)
- bounded responses (`limit` max 200)
- pagination support via `nextCursor`
- `moreAvailable` indicator when additional data exists
- auth required + basic in-memory rate limit

## 2. Parameters

- `preset`: one of `errors|restarts|warnings`
- `range`: one of `15m|1h|6h|24h`
- `limit`: `1..200` (default 100)
- `cursor`: optional continuation cursor from previous response
- `namespace`: optional override (default namespace from config)

## 3. Validation steps

1. Call endpoint with valid token and `preset=errors&range=1h`.
2. Verify HTTP 200 and response fields:
   - `lines[]` with `timestamp`, `message`, `labels`
   - `returned`, `limit`, `moreAvailable`, `nextCursor`
3. Call with invalid preset and verify HTTP 422.
4. Call with `limit=1` and confirm `moreAvailable=true` and non-empty `nextCursor` when extra lines exist.
5. Trigger rate-limit breach (`LOGS_QUICKVIEW_RATE_LIMIT_PER_MIN=1`) and verify second request returns HTTP 429.

## 4. Config

- `LOKI_BASE_URL`
- `LOGS_DEFAULT_NAMESPACE`
- `LOGS_QUICKVIEW_RATE_LIMIT_PER_MIN`

## 5. Evidence files

- `apps/portal/backend/app/main.py`
- `apps/portal/backend/app/logs_quickview.py`
- `apps/portal/backend/tests/test_logs_quickview.py`
- `apps/portal/backend/tests/test_api.py`
