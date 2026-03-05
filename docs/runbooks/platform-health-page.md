# Platform Health Page Runbook

This runbook validates T4.3.10 platform-wide health visibility in the portal.

## 1. Scope

Platform Health page (`/platform-health`) must provide:

- summary cards for service and incident health
- unhealthy services table with service-detail and Grafana links
- latest incident feed with severity/status context
- usable partial-failure behavior when one source is unavailable

## 2. Data source behavior

Load sequence:

1. service registry via `getServicesRegistry()`
2. deployment alert rollups via `getDeploymentHistory()` + `summarizeDeploymentAlerts()`
3. incidents via `GET /api/alerts/active`

Partial failures are surfaced as warning messages on-page.

## 3. Validation steps

1. Open `/platform-health`.
2. Confirm 4 summary cards render:
   - Tracked Services
   - Degraded Services
   - Active Alerts
   - Active Incidents
3. In `Unhealthy Services`, verify each row links to:
   - internal service details (`/services/:serviceId`)
   - external Grafana dashboard (`Open in Grafana`)
4. In `Latest Incidents`, verify severity and status badges render and service-linked incidents have service detail + Grafana links.
5. Simulate incident API failure (or use a disconnected backend) and confirm:
   - page still renders summary and other available sections
   - partial data warning banner is visible
   - incident feed remains empty (no fabricated sample entries)

## 4. Troubleshooting

- No unhealthy services shown:
  - verify service registry has entries and deployment history adapter can resolve recent deployments
- Incident list empty:
  - check `GET /api/alerts/active` response shape
- Grafana links unavailable:
  - verify `VITE_GRAFANA_BASE_URL` and dashboard template settings

## 5. Evidence files

- `apps/portal/frontend/src/pages/platform-health-page.tsx`
- `apps/portal/frontend/src/lib/adapters/platform-health.ts`
- `apps/portal/frontend/src/App.tsx`
- `apps/portal/frontend/src/components/navigation/portal-sidebar.tsx`
- `apps/portal/frontend/src/components/navigation/topbar.tsx`
