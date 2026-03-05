# Platform Health Live Alerts Frontend Runbook

This runbook validates T4.4.11.

## 1. Scope

Frontend must use the live active alerts feed:

- `GET /api/alerts/active`

Behavior:

- platform health page shows latest alert list with severity + start time
- service links are shown when alert includes `serviceId`
- global banner is driven by active alert severity threshold
- dismissing banner is session-scoped and does not hide critical alerts

## 2. Validation steps

1. Open `/platform-health`.
2. Verify Latest Alerts list is populated from live alerts API.
3. Confirm each row shows severity and started timestamp.
4. For rows with `serviceId`, verify `Service details` link routes to `/services/:serviceId`.
5. Confirm `Open in Grafana` link is available when `serviceId` is present.
6. Trigger warning-level alert and verify global banner appears when threshold allows it.
7. Dismiss the banner and reload route; verify warning banner stays hidden for current session.
8. Trigger critical alert and verify banner is visible even after dismissal.

## 3. Evidence files

- `apps/portal/frontend/src/lib/adapters/platform-health.ts`
- `apps/portal/frontend/src/pages/platform-health-page.tsx`
- `apps/portal/frontend/src/App.tsx`
- `apps/portal/frontend/src/lib/incident-alerts.ts`
