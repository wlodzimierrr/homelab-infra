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

## Phase 3: Extract Observability Endpoints

**ID:** R3
**Priority:** Medium
**Risk:** Low
**Estimated lines moved:** ~500

### Description

Move observability endpoints (metrics, alerts, logs, health checks, Prometheus proxy) from `main.py` into `app/api/endpoints/observability.py`.

### Acceptance Criteria

- [ ] Create `app/api/endpoints/observability.py`
- [ ] Move observability handler functions from `main.py`
- [ ] Update route imports
- [ ] All observability tests pass
- [ ] `main.py` line count reduced by ~500 lines

---

## Phase 4: Extract Catalog Endpoints

**ID:** R4
**Priority:** Medium
**Risk:** Medium
**Estimated lines moved:** ~700

### Description

Move catalog/registry endpoints (service list, service detail, project list, catalog reconciliation, sync triggers) from `main.py` into `app/api/endpoints/catalog.py`.

### Acceptance Criteria

- [ ] Create `app/api/endpoints/catalog.py`
- [ ] Move catalog handler functions from `main.py`
- [ ] Update route imports in `app/api/routes/catalog.py` (or equivalent)
- [ ] All catalog tests pass
- [ ] `main.py` line count reduced by ~700 lines

### Notes

Medium risk because catalog handlers have the most shared state (caches, background sync). May need to extract a thin shared-state module or pass dependencies explicitly.

---

## Phase 5: Extract Admin Endpoints

**ID:** R5
**Priority:** Medium
**Risk:** Low
**Estimated lines moved:** ~400

### Description

Move admin/auth endpoints (user management, RBAC checks, admin sync triggers) from `main.py` into `app/api/endpoints/admin.py`.

### Acceptance Criteria

- [ ] Create `app/api/endpoints/admin.py`
- [ ] Move admin handler functions from `main.py`
- [ ] Update route imports
- [ ] All admin tests pass
- [ ] `main.py` line count reduced by ~400 lines

---

## Phase 6: Cleanup main.py

**ID:** R6
**Priority:** Low
**Risk:** Low

### Description

After phases 1-5, `main.py` should be ~1,400 lines containing app setup, middleware, lifespan, and shared utilities. Review what remains and:

- Move any remaining private helpers to the endpoint module that uses them
- Remove dead imports
- Verify `main.py` is purely app bootstrap + middleware + lifespan

### Acceptance Criteria

- [ ] `main.py` contains only app initialization, middleware config, lifespan handlers, and shared utilities
- [ ] No endpoint handler functions remain in `main.py`
- [ ] All tests pass
- [ ] `main.py` is under 600 lines

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
