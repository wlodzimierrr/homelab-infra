# Live-Data-Only Frontend Behavior Runbook

This runbook validates live-data-only adapter behavior after T4.5 cutover.

## 1. Scope

Verify frontend adapters fail closed (no sample JSON/mock fallback).

Primary adapters:

- `release-dashboard`
- `services`
- `platform-health` / incident feed
- `service-health-timeline`
- `deployment-history`

## 2. Expected behavior

- API failures are surfaced as explicit errors/warnings.
- Readiness panels distinguish stale/empty/unavailable sources instead of collapsing into blank tables.
- Monitoring panels distinguish provider failure from valid `no data` results.
- No adapter should fetch sample JSON assets.
- No deployment views should fabricate `*-mock-*` rows.

## 3. Validation steps

1. Deploy frontend against live API sources.
2. Open dashboard/services/platform health pages.
3. Temporarily force one API source to fail (for example return 404/502).
4. Verify UI shows explicit unavailable/error state and does not render sample/mock dataset rows.
5. Confirm Projects and Services pages show live readiness cards/badges for freshness and degradation state.
6. Confirm monitoring views differentiate:
   - provider unreachable
   - upstream unknown
   - no data
7. Confirm browser network panel does not fetch:
   - `/release-dashboard.sample.json`
   - `/services.sample.json`
   - `/platform-health.sample.json`
   - `/service-health-timeline.sample.json`
   - `/service-metrics.sample.json`
8. Confirm deployment views do not show synthetic IDs containing `-mock-`.

## 4. Evidence files

- `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
- `apps/portal/frontend/src/lib/adapters/services.ts`
- `apps/portal/frontend/src/lib/adapters/platform-health.ts`
- `apps/portal/frontend/src/lib/adapters/service-health-timeline.ts`
- `apps/portal/frontend/src/lib/adapters/deployments.ts`
- `apps/portal/frontend/src/pages/projects-page.tsx`
- `apps/portal/frontend/src/pages/services-page.tsx`
- `apps/portal/frontend/src/pages/service-details-page.tsx`
