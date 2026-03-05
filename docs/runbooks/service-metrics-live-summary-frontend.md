# Service Metrics Live Summary Frontend Runbook

This runbook validates T4.4.3 on `/services/:serviceId`.

## 1. Scope

Service metrics cards must use live backend summary endpoint:

- `GET /api/services/:serviceId/metrics/summary?range=...`

Behavior in UI:

- fetch on initial load and on metrics range change (`1h`, `24h`, `7d`)
- show `Last refresh` from `generatedAt`
- show stale marker when refresh age exceeds configured threshold
- show per-card `No data` based on `noData` map without layout shift
- show non-blocking inline error and `Retry metrics` action

## 2. Configuration

- `VITE_METRICS_STALE_AFTER_MINUTES` controls stale threshold (default: `20`).

## 3. Validation steps

1. Open `/services/homelab-api`.
2. Confirm service metrics section loads cards and values from live endpoint.
3. Change range selector across `1h`, `24h`, `7d`; verify cards refetch and update.
4. Confirm `Last refresh` matches backend `generatedAt` and stale marker appears when data ages past threshold.
5. Simulate partial no-data from backend (`noData` true for one metric) and verify only affected cards show `No data`.
6. Simulate endpoint failure and verify inline warning + `Retry metrics` button appears while page stays usable.

## 4. Evidence files

- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/src/lib/adapters/service-metrics.ts`
- `apps/portal/frontend/src/components/service-metric-card.tsx`
- `apps/portal/frontend/src/lib/config.ts`
