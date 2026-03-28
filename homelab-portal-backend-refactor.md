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

## Phase 7: Extract Deployment Helper Cluster — DONE

**ID:** R7
**Priority:** High
**Risk:** Medium
**Estimated lines moved:** ~1,600 (main.py 3,520 → ~1,900)

### Description

Move the deployment-domain private helpers out of `main.py` into `app/helpers/deployment_helpers.py`. These functions are injected as deps into `DeploymentService` and `CatalogService` via the `*ServiceDeps` pattern — the move changes where they're defined, not how they're wired. After this phase, `_build_deployment_service()` and `_build_catalog_service()` in `main.py` become one-line imports rather than giant inline closures.

The cluster spans four functional sub-groups:

**DB / record helpers** (lines 247–403):
`_with_connection`, `_list_deployment_records_for_service`, `_get_deployment_record_by_id`, `_get_active_deployment_lock`, `_upsert_deployment_record_row`, `_reconcile_recent_deployment_activity`, `_maybe_reconcile_recent_deployments`

**GitHub / GHCR / GitOps helpers** (lines 687–1398):
`_extract_version_from_image_ref`, `_github_api_json`, `_build_service_image_ref`, `_build_prod_service_image_ref`, `_extract_sha_from_tag`, `_build_compare_url_for_portal_tags`, `_build_commit_url`, `_parse_ghcr_image_repo`, `_build_package_url_from_image_ref`, `_extract_image_digest`, `_deployment_record_timestamp`, `_select_latest_deployment_info_record`, `_github_package_version_paths`, `_package_version_has_tag`, `_ensure_ghcr_tag_exists`, `_safe_branch_fragment`, `_build_dev_deploy_branch_name`, `_build_prod_promote_branch_name`, `_build_service_rollback_branch_name`, `_extract_image_ref_from_overlay`, `_replace_image_ref_in_overlay`, `_resolve_latest_portal_image_candidate`, `_load_dev_overlay_update_plan`, `_load_promote_to_prod_update_plan`, `_load_service_rollback_update_plan`, `_list_service_rollback_candidates`, `_build_dev_deploy_pr_body`, `_build_promote_to_prod_pr_body`, `_build_service_rollback_pr_body`, `_build_secret_edit_branch_name`, `_build_config_edit_branch_name`, `_build_config_edit_pr_body`, `_build_secret_edit_pr_body`, `_select_preferred_service_row`

**Live runtime / Argo helpers** (lines 1399–1897):
`_normalize_live_sync_status`, `_normalize_live_health_status`, `_release_row_has_meaningful_metadata`, `_coalesce_service_status`, `_list_live_deployments_for_service`, `_load_live_argo_status_for_service`, `_extract_live_deployment_image_ref`, `_extract_live_deployment_health`, `_extract_live_deployment_timestamp`, `_load_live_service_runtime_rows`, `_load_release_rows_for_service`, `_sort_release_rows_by_deployed_at`, `_coalesce_release_string`, `_enrich_release_row_with_live_runtime`, `_enrich_release_rows_with_live_runtime`

**Deployment record response builders** (lines 1898–2323):
`_registry_stale_after_minutes`, `_registry_warning_after_minutes`, `_deployment_history_cache_ttl_seconds`, `_deployment_comparison_window_token`, `_parse_iso_datetime`, `_build_metric_snapshot`, `_format_duration_token`, `_resolve_window_end`, `_expand_observability_query_window`, `_resolve_record_window`, `_query_prometheus_comparison_snapshot`, `_load_metric_snapshots_for_window`, `_load_deployment_metric_snapshots`, `_persist_observability_snapshot_safe`, `_deployment_record_sort_timestamp`, `_build_deployment_record_response`, `_build_deployment_lock_response`

### Acceptance Criteria

- [x] Create `app/helpers/deployment_helpers.py` with all functions listed above
- [x] `_build_deployment_service()` in `main.py` imports helpers from new module (no inline definitions)
- [x] `_build_catalog_service()` in `main.py` imports shared helpers from new module where used
- [x] `py_compile` passes on `main.py` and `app/helpers/deployment_helpers.py`
- [x] `tests/test_api_route_boundaries.py` passes
- [x] `tests/test_deployment_locks.py` passes
- [x] `main.py` line count reduced by ~1,600 lines (3,520 → 1,679, −1,841 lines)

### Notes

`PortalDeployToDevError`, `PortalPromoteToProdError`, `PortalServiceRollbackError` (lines 226–242) should move alongside this cluster since they're used only by deployment helpers and `DeploymentService`. Keep `clear_observability_caches_for_tests()` in `main.py` — it's a test seam for the module-level cache objects.

---

## Phase 8: Extract Observability Helper Cluster — DONE

**ID:** R8
**Priority:** High
**Risk:** Medium
**Estimated lines moved:** ~900 (main.py ~1,900 → ~1,000)
**Actual lines moved:** ~1,068 (main.py 1,679 → 611)

### Description

Move the observability-domain private helpers out of `main.py` into `app/helpers/observability_helpers.py`. These functions are injected into `ObservabilityService` via `ObservabilityServiceDeps`. After this phase `_build_observability_service()` in `main.py` becomes an import + single compose call.

The cluster spans three functional sub-groups:

**Monitoring context resolution** (lines 557–686):
`_resolve_service_monitoring_context`, `_resolve_service_monitoring_metadata`, `_build_service_metrics_probe_queries`, `_query_prometheus_series_present`, `_build_metrics_observability_diagnostics`

**Prometheus / Loki / Alertmanager query wrappers** (lines 2373–2625):
`_query_prometheus_scalar`, `_query_prometheus_range`, `_query_loki_range`, `_query_alertmanager_active_alerts`, `_validate_selected_range`, `_effective_limit`

**Metric / timeline / logs response builders** (lines 2626–3378):
`_build_service_metrics_queries`, `_serialize_metric_trend_points`, `_build_metric_trend_series`, `_build_health_timeline_queries`, `_validate_step_for_range`, `_serialize_metric_snapshot`, `_select_timeline_step_seconds`, `_resolve_deployment_observability_context`, `_build_no_window_metrics_response`, `_build_no_window_timeline_response`, `_build_no_window_logs_response`, `_extract_provider_failure`, `_build_provider_error_metrics_response`, `_build_provider_error_timeline_response`, `_build_provider_error_logs_response`, `_build_deployment_metrics_response`, `_build_deployment_timeline_response`, `_build_deployment_logs_response`

### Acceptance Criteria

- [x] Create `app/helpers/observability_helpers.py` with all functions listed above (1,128 lines)
- [x] `_build_observability_service()` in `main.py` imports helpers from new module (no inline definitions)
- [x] `py_compile` passes on `main.py` and `app/helpers/observability_helpers.py`
- [x] `tests/test_api_route_boundaries.py` passes (11/11)
- [x] `tests/test_catalog_sync_scheduler.py` passes (5/5)
- [x] `main.py` line count reduced by ~1,068 lines (1,679 → 611)

### Notes

The monitoring context helpers (`_resolve_service_monitoring_context`, etc.) depend on catalog DB helpers from R7's `deployment_helpers.py` — import them from there rather than duplicating. The `logger` instance used throughout this cluster should be passed as a parameter or imported from a shared logging setup rather than captured from `main.py`'s module-level `logger`.

---

## Phase 9: Extract Catalog/Registry Helper Cluster and Slim main.py — TODO

**ID:** R9
**Priority:** Medium
**Risk:** Low
**Estimated lines moved:** ~400 (main.py ~1,000 → ~600)

### Description

Move the catalog/registry DB row loaders out of `main.py` into `app/helpers/catalog_helpers.py`, then collapse all four `_build_*_service()` factory functions into `app/services/builders.py` so `main.py` only calls `configure_backend_services()`. After this phase `main.py` should contain only: imports, app init, cache setup, lifespan/middleware, `configure_backend_services()`, `_register_api_routes()`, and the test-seam helpers.

**Catalog / registry DB loaders** (lines 405–554):
`_load_project_rows`, `_load_project_catalog_rows`, `_project_catalog_index`, `_load_service_rows`, `_load_service_catalog_rows`

**Service builder collapse** (lines 2324–3520):
Move `_build_deployment_service()`, `_build_observability_service()`, `_build_catalog_service()`, `_build_scaffold_admin_service()` into `app/services/builders.py` as concrete factory functions that import from `app/helpers/*`. `main.py` becomes: `from app.services.builders import build_all_services; configure_backend_services(build_all_services())`.

### Acceptance Criteria

- [ ] Create `app/helpers/catalog_helpers.py` with DB loaders listed above
- [ ] `app/services/builders.py` contains concrete `_build_*_service()` factories (importing from `app/helpers/*`)
- [ ] `main.py` no longer defines any `_build_*` or `_load_*` functions
- [ ] `py_compile` passes on all touched files
- [ ] `tests/test_api_route_boundaries.py` passes
- [ ] `tests/test_catalog_sync_scheduler.py` passes
- [ ] `tests/test_package_validation.py` passes
- [ ] `main.py` is under 600 lines

### Notes

The `app/helpers/` package needs an `__init__.py`. The `_with_connection()` helper (used by both deployment and catalog helpers) should live in `app/helpers/db.py` and be imported by both, rather than duplicated. The re-exported schema imports (lines 102–166 in current `main.py`) can be removed once all route modules import schemas directly from `app.api.schemas.*` — audit route files before removing.

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

| Order | Ticket | Description                          | Lines moved | Risk    | Status  |
|-------|--------|--------------------------------------|-------------|---------|---------|
| 1     | S1     | Fix prometheus_client import         | ~1          | Trivial | DONE    |
| 2     | S2     | Fix stale monkeypatches              | ~10         | Low     | DONE    |
| 3     | R1     | Extract scaffold endpoints           | ~451        | Low     | DONE    |
| 4     | R2     | Extract deployment endpoints         | ~124        | Low     | DONE    |
| 5     | R3     | Extract observability endpoints      | ~171        | Low     | DONE    |
| 6     | R4     | Extract catalog endpoints            | ~78         | Medium  | PARTIAL |
| 7     | R5     | Extract admin endpoints              | ~53         | Low     | PARTIAL |
| 8     | R6     | Initial main.py cleanup              | ~115        | Low     | PARTIAL |
| 9     | R7     | Extract deployment helper cluster    | ~1,841      | Medium  | DONE    |
| 10    | R8     | Extract observability helper cluster | ~1,068      | Medium  | DONE    |
| 11    | R9     | Extract catalog helpers + slim main  | ~400        | Low     | TODO    |
| 12    | S3     | Review deployment_locks coverage     | ~50         | Low     | TODO    |
