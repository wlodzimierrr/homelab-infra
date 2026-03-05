# Monitoring URL Template Builder Runbook

This runbook validates T4.3.5 template-driven URL generation for Grafana and Loki links.

## 1. Scope

Shared helper in `src/lib/config.ts` now handles:

- template interpolation for monitoring links
- variables: `serviceId`, `namespace`, `environment`, `timeRange` (plus aliases)
- development warnings for missing/unresolved variables
- safe fallback URLs when templates are invalid or variables are missing

## 2. Helper usage

Primary helper:

- `buildMonitoringUrl({ baseUrl, pathTemplate, values, context, fallbackPath })`

Consumers:

- `buildGrafanaDashboardUrl`
- `buildGrafanaLatencyPanelUrl`
- `buildGrafanaErrorPanelUrl`
- `buildLogsUrl`

## 3. Validation steps

1. Run frontend in dev mode and open service detail page.
2. Confirm Grafana and Logs links resolve using shared builders.
3. Temporarily remove a required placeholder value (for example empty `VITE_GRAFANA_BASE_URL`) and confirm:
   - URL returns safe fallback/disabled link
   - dev console shows warning with context prefix `[monitoring-url]`
4. Temporarily break a template with unresolved braces and confirm warning + fallback path behavior.

## 4. Acceptance mapping

- URL interpolation support: satisfied by `buildMonitoringUrl` and config builders.
- Service detail and logs use shared helper: satisfied by `buildGrafana*` + `buildLogsUrl` consumers.
- Invalid/missing variables: dev warnings + safe fallback behavior implemented.

## 5. Evidence files

- `apps/portal/frontend/src/lib/config.ts`
- `apps/portal/frontend/src/pages/service-details-page.tsx`
