# Unhealthy Deployment Highlighting Runbook

This runbook validates T4.3.8 deployment-alert detection and degraded highlighting.

## 1. Scope

Frontend rule helper flags suspicious deployments from configurable thresholds and propagates degraded highlighting into:

- deployment history page (`/services/:serviceId/deployments`)
- service detail page (`/services/:serviceId`)
- service list page (`/services`)

## 2. Detection helper

Central helper module:

- `apps/portal/frontend/src/lib/deployment-alerts.ts`

Default rule inputs include:

- deployment outcome (`failed`, `degraded`, etc.)
- regression score
- error-rate delta
- p95 latency delta
- availability delta (drop)

## 3. Validation steps

1. Open `/services/homelab-api/deployments` and verify suspicious rows are highlighted by `Observability` badges.
2. Set filter to `Regressions only` and confirm suspicious rows remain.
3. Open `/services/homelab-api` and confirm suspicious deployment banner appears when rules trigger.
4. Confirm `StatusCard` health appears degraded when deployment rules mark service as suspicious.
5. Open `/services` and confirm services with suspicious recent deployments show degraded treatment consistently.

## 4. Unit tests

Run frontend tests:

```bash
npm run test:unit
```

Coverage includes deployment alert evaluation and summary logic.

## 5. Evidence files

- `apps/portal/frontend/src/lib/deployment-alerts.ts`
- `apps/portal/frontend/tests/deployment-alerts.test.ts`
- `apps/portal/frontend/src/pages/service-deployments-page.tsx`
- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/src/pages/services-page.tsx`
