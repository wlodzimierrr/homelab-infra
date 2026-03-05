# Project Source Cutover Validation (T4.6.8)

Validate that `/projects` is served from canonical live registry rows and that release/service joins remain consistent.

## Scripted smoke checks

```bash
cd apps/portal/backend
source .venv/bin/activate
python scripts/project_source_cutover_smoke.py \
  --api-base-url http://api.dev.homelab.local \
  --auth-token dev-static-token \
  --env dev \
  --limit 50 \
  --service-check-limit 3
```

Checks performed:

1. `GET /projects` returns canonical rows (`id`, `name`, `environment`) from live source.
2. Smoke fails if legacy seeded/default rows are detected:
   - ids matching `proj*` (`proj`, `proj-*`, `proj_*`)
   - exact legacy ids: `proj-dev`, `proj-prod`
   - legacy name: `Homelab App`
3. `GET /releases?env=...` rows all map to an existing `/projects` `(serviceId, env)` pair.
4. For sample release service IDs, `GET /services/:serviceId/metrics/summary` resolves consistently.

## Manual checks

1. Open `GET /projects` and confirm no seeded/default project IDs.
2. Open `GET /releases?env=dev&limit=50` and verify every row `serviceId` exists in `/projects`.
3. Open dashboard and service detail routes for at least 2 service IDs from `/projects`.
4. Confirm diagnostics endpoint:

```bash
curl -sS -H 'Authorization: Bearer dev-static-token' \
  'http://api.dev.homelab.local/service-registry/diagnostics?env=dev' | jq
```

## Before/after evidence capture (dev cluster)

Capture baseline before cutover migration:

```bash
curl -sS -H 'Authorization: Bearer dev-static-token' http://api.dev.homelab.local/projects | jq
```

Capture post-migration:

```bash
curl -sS -H 'Authorization: Bearer dev-static-token' http://api.dev.homelab.local/projects | jq
python apps/portal/backend/scripts/project_source_cutover_smoke.py \
  --api-base-url http://api.dev.homelab.local \
  --auth-token dev-static-token \
  --env dev
```

Record:

- before: legacy rows present/absent
- after: no legacy rows + smoke PASS
- related test run result (`pytest -q`)

