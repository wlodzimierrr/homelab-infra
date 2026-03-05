# Logs Quick-View Panel Runbook

This runbook validates T4.3.9 logs quick-view behavior on service detail pages.

## 1. Scope

Service detail page includes inline logs quick-view panel with:

- `View logs` action (toggle drawer)
- three scoped presets: `Errors`, `Restarts`, `Warnings`
- one-click deep links to Grafana/Loki for each preset
- `Open full logs` deep link preserving equivalent context

## 2. Context scoping

Preset links are scoped by service context:

- `serviceId`
- `namespace`
- `app_label`
- `timeRange`

Current defaults in UI:

- namespace: `default`
- app label: current `serviceId`
- range: `6h`

## 3. Validation steps

1. Open `/services/homelab-api`.
2. In Quick Links -> Logs, click `View logs`.
3. Confirm panel renders presets: Errors, Restarts, Warnings.
4. Switch presets and verify query preview updates.
5. Click `Open in Grafana` for each preset and verify deep link opens Loki with same service scope.
6. Click `Open full logs` and verify equivalent scoped context is preserved.

## 4. Template notes

`VITE_LOKI_LOGS_PATH_TEMPLATE` supports optional placeholders for preset/query contexts if desired by environment template:

- `{preset}` / `{{preset}}`
- `{query}` / `{{query}}`

If template does not include these placeholders, links still open with baseline service-scoped context.

## 5. Evidence files

- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/src/lib/config.ts`
