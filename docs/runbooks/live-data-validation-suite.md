# Live-Data Validation Suite (T4.5.8)

This runbook validates that portal UI flows are backed by live API responses and not sample/mock paths.

## Scope

- Dashboard release data (`/api/releases`)
- Service detail metrics + health timeline
- Logs quick-view endpoint
- Platform health alerts feed
- Sample asset reachability checks

## Preconditions

- Portal frontend deployed and reachable.
- Backend API reachable.
- Valid auth context for API requests (Bearer token or auth cookie).
- Known service id for checks (for example `homelab-api`).

## Scripted Smoke Check

From `apps/portal/frontend`:

```bash
API_BASE_URL="http://portal.dev.homelab.local/api" \
PORTAL_BASE_URL="http://portal.dev.homelab.local" \
SERVICE_ID="homelab-api" \
AUTH_TOKEN="dev-static-token" \
npm run test:live-smoke
```

Alternative auth (cookie):

```bash
API_BASE_URL="http://portal.dev.homelab.local/api" \
PORTAL_BASE_URL="http://portal.dev.homelab.local" \
SERVICE_ID="homelab-api" \
AUTH_COOKIE="_oauth2_proxy=<cookie-value>" \
npm run test:live-smoke
```

The script checks:

1. `GET /releases?limit=20`
2. `GET /services/:serviceId/metrics/summary?range=24h`
3. `GET /services/:serviceId/health/timeline?range=24h&step=5m`
4. `GET /services/:serviceId/logs/quickview?preset=errors&range=1h&limit=50`
5. `GET /alerts/active`

And asserts:

- responses are HTTP 200 JSON
- response payloads do not include mock/sample markers (`-mock-`, `sample_fallback`)
- sample JSON assets are not directly reachable from deployed UI paths:
  - `/release-dashboard.sample.json`
  - `/services.sample.json`
  - `/platform-health.sample.json`
  - `/service-health-timeline.sample.json`
  - `/service-metrics.sample.json`

## Manual UI Validation

1. Open `/dashboard`.
2. Confirm release rows render with live values and no mock row IDs.
3. Open `/services/:serviceId`.
4. Confirm metric cards load live summary values and timeline/log quick-view render.
5. Open `/platform-health`.
6. Confirm alert feed is populated from live alerts or shows non-blocking warning (without fabricated rows).
7. Open browser Network tab and verify sample JSON files are not requested during normal navigation.

## Failure Triage

- If response is HTML/sign-in page:
  - session/auth gateway issue; re-authenticate or pass valid auth token/cookie.
- If `sample`/`mock` markers appear:
  - inspect adapter fallback flags and strict live-data mode.
- If sample asset path returns HTTP 200 in deployed environment:
  - deployment serving unintended static files; remove or block sample assets.

