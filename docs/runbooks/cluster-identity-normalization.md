# Cluster Identity Normalization (T4.5.7)

This runbook standardizes service identity keys used by release, metrics, and logs joins.

## Goal

Ensure all UI/backend joins use canonical `serviceId` values and avoid mixed formats like:
- human labels: `Browser Project`
- runtime names: `Portal Project`
- canonical IDs: `browser-project`, `portal-project`

## Canonicalization Rule

Frontend canonical service IDs are normalized as:
1. trim
2. lowercase
3. replace non `[a-z0-9]` groups with `-`
4. collapse duplicate `-`
5. trim leading/trailing `-`

Examples:
- `Browser Project` -> `browser-project`
- `Portal_Project` -> `portal-project`
- ` homelab-api ` -> `homelab-api`

## Migration Steps

1. Inspect current project rows:
   - `GET /api/projects`
   - capture `id`, `name`, `environment`

2. Build normalization mapping from project names:
   - source: `name`
   - target: canonicalized `serviceId`
   - keep display labels in `name`; only joins/routes use canonical key

3. Validate release rows normalize to canonical IDs:
   - `GET /api/releases?limit=50`
   - confirm dashboard rows route to `/services/:serviceId` using canonical IDs

4. Check join mismatch diagnostics:
   - open release dashboard
   - verify warning panel shows explicit unmatched keys in format:
     - `serviceId|serviceName|environment`

5. Validate downstream joins:
   - open one normalized service route
   - verify metrics summary, logs quick-view, and Grafana links still load for the same `serviceId`

## Rollback

If normalization breaks a service route:
1. Inspect unmatched key diagnostics on dashboard.
2. Verify backend/release metadata emits expected service key.
3. Add/adjust registry naming so canonical service key matches release+monitoring metadata.

