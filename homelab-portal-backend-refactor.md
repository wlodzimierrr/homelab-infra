# Portal Backend Refactor Plan

## Problem Statement

`apps/portal/backend/app/main.py` is 4,480 lines with 173 functions (125 private helpers, 48 public endpoint handlers). It acts as an oversized integration layer — every endpoint handler lives here even though the service layer, schema layer, and route registration are properly separated. The file is the single biggest obstacle to maintainability.

## What's Fine (Do Not Touch)

- **Service layer** (`app/services/`) — clean domain boundaries, proper dependency injection
- **Schema layer** (`app/api/schemas/`) — well-separated request/response models
- **Route registration** (`app/api/routes/`) — thin dispatch, correct pattern
- **Scaffold engine** (`app/services/scaffold_admin_service.py`) — cohesive, testable
- **Test coverage** — good patterns, tests are co-located with domains

## Strategy

Extract endpoint handler functions from `main.py` into `app/api/endpoints/` modules, one per domain. Each module gets the handler functions that currently live in main.py. Routes in `app/api/routes/` already point to `main.<handler>` — update them to point to `endpoints.<module>.<handler>`. No service layer changes. No new abstractions.

---

## Phase 1: Extract Scaffold Endpoints — DONE

**ID:** R1
**Priority:** High
**Risk:** Low
**Estimated lines moved:** ~800
**Actual lines moved:** ~451 (main.py 4,480 → 4,029)

### Description

Move all scaffold-related endpoint handlers from `main.py` into `app/api/endpoints/scaffold.py`. These are the handlers called by routes in `app/api/routes/scaffold.py`.

### Acceptance Criteria

- [x] Create `app/api/endpoints/scaffold.py` (463 lines)
- [x] Move scaffold handler functions (scaffold preview/submit/list, adopt, validate migration, consolidate migration, update hostname) from `main.py` to the new module
- [x] Update `app/api/routes/scaffold.py` imports to point to `app.api.endpoints.scaffold`
- [x] All existing scaffold tests pass without modification (or with import-path-only changes)
- [x] `main.py` line count reduced by ~451 lines (4,480 → 4,029)

### Additional work

- Extracted auth dependencies (`require_admin`, `get_current_user`, `require_bearer_token`) from `main.py` into `app/api/deps/auth.py` to break circular imports between endpoint modules and main.py
- Updated `app/api/deps/__init__.py` to re-export auth deps
- Updated `tests/test_api_route_boundaries.py` and `tests/test_api_scaffold.py` to reference new module paths
- Test suite: 19 failed / 386 passed (unchanged from baseline — zero regressions)

### Notes

Scaffold handlers are the most self-contained — they delegate almost entirely to `ScaffoldAdminService`. Config/secret handlers still use `_get_scaffold_admin_service()` from main.py — they'll move in a later phase.

---

## Phase 2: Extract Deployment Endpoints — DONE

**ID:** R2
**Priority:** High
**Risk:** Low
**Estimated lines moved:** ~600
**Actual lines moved:** ~124 (main.py 4,029 → 3,905)

### Description

Move deployment-related endpoint handlers (reconcile, get, create, cancel, deploy-to-dev, promote-to-prod, rollback candidates, rollback, portal rollback, deployments list, deployment info) from `main.py` into `app/api/endpoints/deployments.py`.

### Acceptance Criteria

- [x] Create `app/api/endpoints/deployments.py` (172 lines)
- [x] Move 11 deployment handler functions from `main.py`
- [x] Update route imports in `app/api/routes/deployments.py`
- [x] All deployment tests pass
- [x] `main.py` line count reduced by ~124 lines (4,029 → 3,905)

### Notes

Handlers are thin wrappers delegating to `DeploymentService`. The `reconcile_deployments` handler uses a lazy import for the reconciliation function and cache that remain in main.py. Config/secret handlers still live in main.py (scaffold admin service domain). Test suite: 19 fail / 386 pass (unchanged).

---

## Phase 3: Extract Observability Endpoints — DONE

**ID:** R3
**Priority:** Medium
**Risk:** Low
**Estimated lines moved:** ~500
**Actual lines moved:** ~171 (main.py 3,905 → 3,734)

### Description

Move 12 observability endpoint handlers (deployment observability, provider diagnostics, metrics summary/trends/legacy, health timeline, alerts, incidents, releases, release dashboard, logs quickview) from `main.py` into `app/api/endpoints/observability.py`.

### Acceptance Criteria

- [x] Create `app/api/endpoints/observability.py` (224 lines)
- [x] Move 12 observability handler functions from `main.py`
- [x] Update route imports
- [x] All observability tests pass
- [x] `main.py` line count reduced by ~171 lines (3,905 → 3,734)

### Notes

All handlers are thin delegators to `ObservabilityService`. Test suite: 19 fail / 386 pass (unchanged).

---

## Phase 4: Extract Catalog Endpoints — PARTIAL

**ID:** R4
**Priority:** Medium
**Risk:** Medium
**Estimated lines moved:** ~700
**Actual lines moved:** ~78 (main.py 3,734 → 3,656)

### Description

Move catalog/registry endpoints (service list, service detail, project list, catalog reconciliation, sync triggers) from `main.py` into `app/api/endpoints/catalog.py`.

### Acceptance Criteria

- [x] Create `app/api/endpoints/catalog.py`
- [x] Move catalog handler functions from `main.py`
- [x] Update route imports in `app/api/routes/catalog.py` (or equivalent)
- [ ] All catalog tests pass
- [x] `main.py` line count reduced by 78 lines (3,734 → 3,656)

### Notes

Handlers now live in `app/api/endpoints/catalog.py` and follow the same lazy service-builder pattern as the deployment/observability extractions, so no new shared-state abstraction was required for this phase.

Targeted non-`TestClient` coverage passed:

- `tests/test_service_registry_sync.py`
- `tests/test_catalog_sync_scheduler.py`
- `tests/test_catalog_reconciliation.py`
- `tests/test_projects_backfill.py`
- `tests/test_live_catalog_validation.py`
- `tests/test_api_route_boundaries.py`
- `tests/test_deployment_locks.py`

Client-based API tests are still blocked in this sandbox because `fastapi.testclient.TestClient(app).__enter__()` hangs before the first request. That reproduces in `tests/test_api_catalog.py` and also in a non-catalog test (`tests/test_api_auth.py::test_login_success`), so this looks like a broader test harness/runtime issue rather than a catalog-route regression.

---

## Phase 5: Extract Admin Endpoints — PARTIAL

**ID:** R5
**Priority:** Medium
**Risk:** Low
**Estimated lines moved:** ~400
**Actual lines moved:** ~53 (main.py 3,656 → 3,603)

### Description

Move admin/auth endpoints (user management, RBAC checks, admin sync triggers) from `main.py` into `app/api/endpoints/admin.py`.

### Acceptance Criteria

- [x] Create `app/api/endpoints/admin.py`
- [x] Move admin handler functions from `main.py`
- [x] Update route imports
- [ ] All admin tests pass
- [x] `main.py` line count reduced by 53 lines (3,656 → 3,603)

### Notes

This phase extracted the remaining auth/admin endpoint wrappers into `app/api/endpoints/admin.py`: `login`, `get_service_config`, `request_portal_set_config`, and `request_portal_set_secret`.

Verified locally:

- `py_compile` on the touched admin/auth route and endpoint files
- `tests/test_api_route_boundaries.py` passes

Client-based auth/admin API tests are still blocked by the same sandbox/runtime issue seen in Phase 4: `fastapi.testclient.TestClient(app).__enter__()` hangs before the first request. That reproduces for `tests/test_api_auth.py` and `tests/test_api_admin_mutations.py`, so this remains a broader `TestClient`/AnyIO thread-interop problem rather than a Phase 5 route extraction regression.

---

## Phase 6: Cleanup main.py — PARTIAL

**ID:** R6
**Priority:** Low
**Risk:** Low
**Actual lines moved:** ~115 (main.py 3,603 → 3,488)

### Description

After phases 1-5, `main.py` should be ~1,400 lines containing app setup, middleware, lifespan, and shared utilities. Review what remains and:

- Move any remaining private helpers to the endpoint module that uses them
- Remove dead imports
- Verify `main.py` is purely app bootstrap + middleware + lifespan

### Acceptance Criteria

- [ ] `main.py` contains only app initialization, middleware config, lifespan handlers, and shared utilities
- [x] No endpoint handler functions remain in `main.py`
- [ ] All tests pass
- [ ] `main.py` is under 600 lines

### Notes

This pass extracted the final public route handlers (`/health` and `/metrics`) into `app/api/endpoints/system.py`, updated `app/api/routes/system.py`, and removed the corresponding handler definitions from `main.py`.

It also cleaned up a large batch of dead imports from `main.py`. Some schema imports remain intentionally even though static lint sees them as unused, because the split route modules still resolve response models through `main_module.<SchemaName>` during route registration.

Verified locally:

- `py_compile` on the touched `main.py` and system endpoint files
- `tests/test_api_route_boundaries.py`
- `tests/test_catalog_sync_scheduler.py`
- `tests/test_package_validation.py`

Status after this pass:

- `main.py` no longer contains public endpoint handlers
- `main.py` is still much larger than the target because the deployment/catalog/observability helper clusters and composition callbacks still live there
- Full API test coverage remains blocked by the existing `TestClient`/AnyIO thread-interop hang in this sandbox

---

## Secondary Issues

### S1: Fix Missing prometheus_client Import

**Priority:** Critical (blocks test suite)
**Risk:** Trivial

`main.py` line 16 imports `CONTENT_TYPE_LATEST, generate_latest` from `prometheus_client` but `Counter` and `Histogram` are used at lines 288-293 without being imported. This breaks the test conftest import chain.

**Fix:** Add `Counter, Histogram` to the existing `prometheus_client` import.

---

### S2: Fix Stale Monkeypatches in test_api_admin_mutations.py

**Priority:** High
**Risk:** Low

Some monkeypatch targets in `test_api_admin_mutations.py` still reference `app.main.*` for functions that now live in the service layer. The cluster sync and gitops sync patches were partially fixed but should be verified end-to-end.

**Fix:** Audit all `monkeypatch.setattr` calls in the file and update targets to match current module paths.

---

### S3: Review deployment_locks Test Coverage

**Priority:** Low
**Risk:** Low

The `deployment_locks` mechanism in main.py has minimal test coverage. As deployment endpoints are extracted (R2), add unit tests for lock acquisition/release edge cases.

---

## Execution Order

| Order | Ticket | Lines | Risk   |
|-------|--------|-------|--------|
| 1     | S1     | ~1    | Trivial|
| 2     | S2     | ~10   | Low    |
| 3     | R1     | ~800  | Low    |
| 4     | R2     | ~600  | Low    |
| 5     | R3     | ~500  | Low    |
| 6     | R4     | ~700  | Medium |
| 7     | R5     | ~400  | Low    |
| 8     | R6     | —     | Low    |
| 9     | S3     | ~50   | Low    |
