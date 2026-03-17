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

## 2a. Regression thresholds (current defaults)

All thresholds live in `DEFAULT_DEPLOYMENT_ALERT_THRESHOLDS` in
`apps/portal/frontend/src/lib/deployment-alerts.ts`.

| Metric | Warning | Critical | Notes |
|---|---|---|---|
| Regression score | ≥ 0.8 | ≥ 1.5 | Composite score; critical means multiple signals fired simultaneously |
| Error-rate delta | ≥ 0.4 pp | ≥ 1.0 pp | Percentage-point change vs pre-deploy window |
| P95 latency delta | ≥ 80 ms | ≥ 180 ms | Absolute ms increase vs pre-deploy window |
| Availability drop | ≥ 0.12 pp | ≥ 0.3 pp | Positive value = drop (availability delta is negated before comparing) |
| Suspicious outcomes | — | `failed`, `degraded`, `error` | Outcome string match; immediately triggers critical |

Alert levels per deployment:
- **none** — all thresholds below warning; outcome is normal
- **warning** — at least one metric exceeded the warning threshold
- **critical** — at least one metric exceeded the critical threshold, or outcome is suspicious

Alert propagation:
- Deployment history timeline: `ImpactBadge` per row; filter by `Regressions only` or `Missing samples`
- Service detail overview: compact per-env impact row (outcome + badge + deltas); suspicious banner if any recent deployment triggered an alert
- Services list: `Deploy: warning` or `Deploy: critical` badge per service row (based on last 3 deployments)
- Platform health dashboard: `summarizeDeploymentAlerts` aggregates across all services

**To adjust thresholds:** edit the `DEFAULT_DEPLOYMENT_ALERT_THRESHOLDS` constant directly and rebuild the frontend. Thresholds are frontend-only and do not require a backend change. Restart or redeploy `homelab-web` after the build to apply.

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
