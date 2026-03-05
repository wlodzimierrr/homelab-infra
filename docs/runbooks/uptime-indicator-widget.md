# Uptime Indicator Widget Runbook

This runbook validates T4.3.3 shared uptime indicator behavior.

## 1. Scope

Shared `UptimeIndicator` component is reused in:

- service detail page (`/services/:serviceId`)
- services list page (`/services`)

The widget displays:

- 24h and 7d uptime values
- severity mapped from centralized thresholds
- `Last refresh` timestamp
- stale-data marker when data age exceeds threshold
- explicit loading and no-data states

## 2. Central mapping

Central mapping module:

- `apps/portal/frontend/src/lib/uptime-status.ts`

Default thresholds:

- healthy: `>= 99.9`
- warning: `>= 99.0` and `< 99.9`
- critical: `< 99.0`
- stale after: `20 minutes`

## 3. Unit tests

Run in frontend project:

```bash
npm run test:unit
```

Covered functions:

- `classifyUptime`
- `isMetricStale`
- `mergeUptimeSeverities`

## 4. Validation steps

1. Open `/services/homelab-api` and confirm uptime widget renders with 24h/7d values.
2. Open `/services` and confirm the same widget appears in service rows.
3. Confirm missing metrics render `No data` without layout break.
4. Confirm stale marker appears when `lastRefreshedAt` is older than threshold.

## 5. Evidence files

- `apps/portal/frontend/src/components/uptime-indicator.tsx`
- `apps/portal/frontend/src/lib/uptime-status.ts`
- `apps/portal/frontend/tests/uptime-status.test.ts`
- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/src/pages/services-page.tsx`
