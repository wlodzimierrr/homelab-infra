# Global Incident Banner and Alert Badges Runbook

This runbook validates T4.3.11 incident visibility in the portal.

## 1. Scope

Portal behavior includes:

- app-wide incident banner on main routes when active incidents meet severity threshold
- per-service alert count badges in services list and service detail views
- dismiss-for-session behavior for non-critical banners
- critical incident visibility preserved even after dismiss attempts

## 2. Data source behavior

Incident signals load from:

1. `GET /api/monitoring/incidents`
2. fallback `apps/portal/frontend/platform-health.sample.json`

Banner severity threshold is configured by `VITE_INCIDENT_BANNER_MIN_SEVERITY` (`warning` default).

## 3. Validation steps

1. Open `/dashboard` and verify banner appears when active incidents satisfy configured threshold.
2. Dismiss the banner (non-critical) and navigate to `/projects` and `/services`; verify it stays hidden for this session.
3. Verify critical incidents still show banner even if a previous warning banner was dismissed.
4. Open `/services`; confirm rows with active incidents show alert count badges with severity color mapping.
5. Open `/services/:serviceId`; confirm service header shows matching alert count badge for that service.

## 4. Troubleshooting

- Banner never appears:
  - verify incidents include `status: active`
  - verify `VITE_INCIDENT_BANNER_MIN_SEVERITY` is not set too high
- Service badge missing:
  - verify incident has `serviceId` matching portal service id
- Dismiss state not sticky:
  - verify browser `sessionStorage` is writable

## 5. Evidence files

- `apps/portal/frontend/src/App.tsx`
- `apps/portal/frontend/src/components/incident-banner.tsx`
- `apps/portal/frontend/src/lib/incident-alerts.ts`
- `apps/portal/frontend/src/pages/services-page.tsx`
- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/tests/incident-alerts.test.ts`
