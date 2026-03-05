# Projects Backfill Migration to Canonical Service Registry

Run this once to migrate legacy `projects` rows to canonical `service_registry`.

## Prerequisites

- Backend virtualenv activated (`apps/portal/backend/.venv`)
- `DATABASE_URL` points to target DB
- Alembic at head (`alembic upgrade head`)

## Dry-run (report only)

```bash
cd apps/portal/backend
source .venv/bin/activate
python scripts/migrate_projects_to_service_registry.py
```

Dry-run outputs:

- summary counts (`insert`, `update`, `noop`)
- per-row plan (`legacyProjectId`, canonical `serviceId`, `mappingRule`)
- rollback SQL preview

## Apply migration + write rollback file

```bash
python scripts/migrate_projects_to_service_registry.py \
  --apply \
  --rollback-file /tmp/projects-backfill-rollback.sql
```

## Rollback

If rollback is required:

```bash
psql "$DATABASE_URL" -f /tmp/projects-backfill-rollback.sql
```

## Post-migration validation

1. Check canonical projects projection:

```bash
curl -sS -H 'Authorization: Bearer dev-static-token' \
  http://api.dev.homelab.local/projects | jq
```

2. Check release joins still resolve canonical service IDs:

```bash
curl -sS -H 'Authorization: Bearer dev-static-token' \
  'http://api.dev.homelab.local/releases?env=dev&limit=50' | jq
```

3. Verify frontend links/routes still resolve using canonical `serviceId` values:
- Release dashboard service links
- Service details route
- Metrics/logs quick links

