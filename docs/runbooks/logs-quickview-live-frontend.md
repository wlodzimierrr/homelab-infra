# Logs Quick-View Live Frontend Runbook

This runbook validates T4.4.9 on `/services/:serviceId`.

## 1. Scope

Logs quick-view panel must use live backend endpoint:

- `GET /api/services/:serviceId/logs/quickview?preset=...&range=...`

Behavior in UI:

- loads presets `errors`, `restarts`, `warnings`
- supports range selector (`15m`, `1h`, `6h`, `24h`)
- shows non-blocking loading, empty, and error states
- keeps `Open full logs` deep link usable in all states
- preserves query context and selected range in deep links

## 2. Validation steps

1. Open `/services/homelab-api`.
2. In Quick Links, click `View logs` to open the drawer.
3. Switch presets across `Errors`, `Restarts`, `Warnings`; verify content refetches.
4. Switch range across `15m`, `1h`, `6h`, `24h`; verify content refetches and deep link updates.
5. Verify loading state appears while request is in flight.
6. Simulate no matching logs and verify explicit empty state appears.
7. Simulate endpoint failure and verify inline error + `Retry logs` appears while drawer stays usable.
8. Verify `Open full logs` remains clickable and opens Grafana/Loki with matching preset/range context.

## 3. Evidence files

- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/src/lib/adapters/logs-quickview.ts`
- `apps/portal/frontend/src/lib/config.ts`
