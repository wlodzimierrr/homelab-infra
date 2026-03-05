# Release Traceability Endpoint Runbook

This runbook validates backend ticket T4.4.6.

## 1. Scope

Endpoint:

- `GET /api/releases?env=dev&limit=50`

Row shape includes:

- `serviceId`, `env`, `commitSha`, `imageRef`, `deployedAt`
- `argo`: `{ appName, syncStatus, healthStatus, revision }`
- `drift`: `{ isDrifted, expectedRevision, liveRevision }`

Server-side filters:

- `env`
- `serviceId`
- `limit` (`1..200`)

## 2. Deterministic drift rule

A row is drifted when the first matching condition is true:

1. `syncStatus == out_of_sync`
2. `expectedRevision` and `liveRevision` both exist and differ
3. `expectedImageRef` and `liveImageRef` both exist and differ

Otherwise, `isDrifted=false`.

## 3. Upstream metadata and unknown handling

Metadata sources:

- `RELEASE_CI_METADATA_JSON`
- `RELEASE_ARGO_METADATA_JSON`

When metadata is missing, endpoint still returns rows with explicit unknown values (`syncStatus=unknown`, `healthStatus=unknown`, nullable revision/image/commit fields).

## 4. Validation steps

1. Call `/api/releases?env=dev&limit=50` with valid auth token.
2. Verify response rows include nested `argo` and `drift` objects.
3. Verify filtering by `serviceId` and `env` works.
4. Provide mismatched expected/live revision in metadata and confirm `isDrifted=true`.
5. Remove metadata and verify rows still return with unknown fields.

## 5. Evidence files

- `apps/portal/backend/app/main.py`
- `apps/portal/backend/app/release_traceability.py`
- `apps/portal/backend/tests/test_release_traceability.py`
- `apps/portal/backend/tests/test_api.py`
