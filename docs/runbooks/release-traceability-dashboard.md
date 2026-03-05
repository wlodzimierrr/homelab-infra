# Release Traceability Dashboard Runbook

This runbook validates T4.3.1 in the portal dashboard (`/dashboard`).

## 1. Purpose

The dashboard provides release traceability across:

- Git commit SHA
- Deployed image tag/reference
- Argo CD sync state
- Explicit drift indicator

## 2. Data sources and fallback behavior

Dashboard loading order:

1. `GET /api/releases?limit=50` (live traceability endpoint)
2. `GET /api/projects` mapped to minimal release rows (sync + health + environment)
3. Frontend fallback file `apps/portal/frontend/release-dashboard.sample.json` only when enabled in dev mode or with `VITE_ENABLE_RELEASE_SAMPLE_FALLBACK=true`

Notes:

- Full commit/image links are available when release metadata is present (API or sample data).
- Drift is treated as true when any of these signals are present:
  - `sync == out_of_sync`
  - deployed image differs from desired image
  - deployed commit differs from desired commit
  - explicit backend `drift: true`

## 3. Validation steps

1. Open portal dashboard (`/dashboard`).
2. Confirm each row includes service, commit, image, Argo sync, drift, and deployed timestamp.
3. Confirm commit SHA cell opens a commit URL when link metadata exists.
4. Confirm image cell opens registry/package URL when link metadata exists.
5. Confirm drift badge shows `Drift detected` for out-of-sync or mismatched desired/deployed metadata.
6. Use `Drift only` filter and verify non-drifted rows are hidden.
7. Confirm data-source badge is shown (`Live`, `Fallback: Projects`, or `Fallback: Sample`).

## 4. Troubleshooting

- Empty table with no error:
  - Verify API auth is valid.
  - Verify fallback sample file exists and is valid JSON.
- Missing commit/image links:
  - Confirm release payload includes `commitUrl` and `imageUrl`.
- Drift unexpectedly always `In sync`:
  - Check `sync` normalization and desired/deployed fields in payload.

## 5. Evidence for T4.3.1

- UI implemented in `apps/portal/frontend/src/pages/dashboard-page.tsx`
- Adapter implemented in `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
- Fallback dataset in `apps/portal/frontend/release-dashboard.sample.json`
