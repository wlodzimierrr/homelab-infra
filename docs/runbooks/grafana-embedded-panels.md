# Grafana Embedded Panels Runbook

This runbook validates T4.3.4 for service detail page Grafana embeds.

## 1. Scope

Service detail page (`/services/:serviceId`) includes two embedded Grafana panels:

- P95 latency trend
- Error-rate trend

Each panel must provide:

- in-page embed rendering
- non-blocking fallback state when embed fails
- deep link button (`Open in Grafana`) preserving service and time-range context

## 2. Configuration

Required frontend runtime variables:

- `VITE_GRAFANA_BASE_URL`
- `VITE_GRAFANA_LATENCY_PANEL_PATH_TEMPLATE`
- `VITE_GRAFANA_ERROR_PANEL_PATH_TEMPLATE`
- `VITE_GRAFANA_DASHBOARD_PATH_TEMPLATE` (for deep-link target)

Default templates are provided in `src/lib/config.ts` and include `serviceId` and `timeRange` interpolation.
The default homelab setup now points at the provisioned Grafana dashboard UID `portal-service-embeds`, with:

- panel `2` for P95 latency
- panel `3` for error rate
- URL variables for `serviceId`, `namespace`, `appLabel`, `environment`, and `argoAppName`

## 3. Validation steps

1. Open `/services/homelab-api`.
2. Confirm `Latency & Error Trends` section renders two panels.
3. Confirm each panel has `Open in Grafana` action.
4. Click `Open in Grafana` and verify URL includes service variable and time-range (`6h` default in UI).
5. Verify the resolved URL targets `portal-service-embeds` and includes `var-service`, `var-namespace`, `var-app`, and `var-env`.
6. Simulate embed failure (for example invalid panel template) and verify:
   - panel displays fallback message
   - page remains usable (non-blocking)
   - fallback still offers working deep link button

## 4. Troubleshooting

- Panels remain in fallback state:
  - verify Grafana allows embedding (`allow_embedding = true`)
  - verify dashboard UID `portal-service-embeds` and panel IDs `2` / `3` in templates
  - verify Grafana auth/session allows iframe access
- Deep link opens wrong scope:
  - check `{serviceId}`, `{namespace}`, `{appLabel}`, `{environment}`, `{argoAppName}`, and `{timeRange}` placeholders in dashboard path template

## 5. Evidence files

- `apps/portal/frontend/src/components/grafana-embed-panel.tsx`
- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/src/lib/config.ts`
- `workloads/environments/dev/workloads/monitoring-app.yaml`
