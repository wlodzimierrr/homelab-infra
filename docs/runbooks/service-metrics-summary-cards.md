# Service Metrics Summary Cards Runbook

This runbook validates T4.3.2 for `/services/:serviceId`.

## 1. Scope

Service detail page displays four summary cards:

- Uptime % (24h)
- P95 latency (ms)
- Error rate %
- Restart count

Each card shows:

- current value (or `No data`)
- severity state (`healthy`, `warning`, `critical`, or `No data`)
- `Last refresh` timestamp

## 2. Data source behavior

Load order for metrics:

1. `GET /api/services/{serviceId}/metrics-summary`
2. local fallback file: `apps/portal/frontend/service-metrics.sample.json`
3. safe no-data fallback (all metric values undefined)

## 3. Threshold mapping

Severity thresholds currently configured in service detail page:

- Uptime (higher is better):
  - healthy: `>= 99.9`
  - warning: `>= 99.0` and `< 99.9`
  - critical: `< 99.0`
- P95 latency (lower is better):
  - healthy: `<= 250 ms`
  - warning: `> 250 ms` and `<= 500 ms`
  - critical: `> 500 ms`
- Error rate (lower is better):
  - healthy: `<= 1%`
  - warning: `> 1%` and `<= 3%`
  - critical: `> 3%`
- Restart count (lower is better):
  - healthy: `<= 1`
  - warning: `> 1` and `<= 3`
  - critical: `> 3`

## 4. Validation steps

1. Open one known service page, for example `/services/homelab-api`.
2. Confirm all four metric cards are visible.
3. Confirm each card includes a `Last refresh` value.
4. Confirm cards use state tones consistent with existing status badges.
5. Open a service missing sample/API metrics and confirm cards render `No data` without layout break.

## 5. Evidence files

- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/src/components/service-metric-card.tsx`
- `apps/portal/frontend/src/lib/adapters/service-metrics.ts`
- `apps/portal/frontend/service-metrics.sample.json`
