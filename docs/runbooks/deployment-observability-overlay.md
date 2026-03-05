# Deployment Observability Overlay Runbook

This runbook validates T4.3.7 deployment history observability overlays.

## 1. Scope

Deployment history page (`/services/:serviceId/deployments`) includes post-deploy metric snapshots:

- error-rate before/after and delta
- p95 latency before/after and delta
- availability before/after and delta

Page also supports:

- explicit `Comparison unavailable` state for missing windows
- filter by regressions/missing comparisons
- sort by `Worst impact first`

## 2. Data behavior

Adapter source:

- `apps/portal/frontend/src/lib/adapters/deployments.ts`

Current strategy:

1. backend deployments endpoint (`/services/{serviceId}/deployments`)
2. adapter normalizes any metric snapshot fields if present
3. fallback mock history includes synthetic snapshots and missing-window rows

## 3. Validation steps

1. Open `/services/homelab-api/deployments`.
2. Confirm rows include before/after indicators for error rate and latency.
3. Confirm at least one row renders explicit unavailable comparison state.
4. Set filter to `Regressions only` and verify non-regression rows are hidden.
5. Set sort to `Worst impact first` and verify highest regression impact rows appear first.

## 4. Troubleshooting

- All rows show unavailable comparisons:
  - verify backend snapshot fields or fallback mock generation
- Sort looks unchanged:
  - verify rows have non-zero regression scores
- Missing overlays:
  - check adapter normalization for snapshot key names

## 5. Evidence files

- `apps/portal/frontend/src/lib/adapters/deployments.ts`
- `apps/portal/frontend/src/pages/service-deployments-page.tsx`
