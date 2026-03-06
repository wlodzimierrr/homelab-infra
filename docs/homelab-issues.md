# Homelab Portal Issues

Last updated: 2026-03-06

## Current dev state

Observed against `http://api.dev.homelab.local` on 2026-03-06:

- API process is healthy: `GET /health` returns `200 {"status":"ok"}`.
- `/projects` returns GitOps-backed rows for `homelab-api` and `homelab-web`.
- `/services` returns cluster-backed rows and `/service-registry/diagnostics` reports `freshness.state="fresh"`.
- `POST /service-registry/sync?source=gitops_apps&env=dev` returns discovered/upserted rows with no `sourceFailures`.
- `POST /service-registry/sync?source=cluster_services&env=dev` returns discovered/upserted rows with no `sourceFailures`.
- `GET /monitoring/providers/diagnostics` reports `prometheus`, `loki`, and `alertmanager` all `healthy`.
- `GET /alerts/active` and `GET /monitoring/incidents` return healthy `providerStatus`.
- Latest scheduled `homelab-api-catalog-sync` Job completed successfully on 2026-03-06 and emitted `catalog_sync_source_result` / `catalog_sync_run_result` log lines for both `gitops_apps` and `cluster_services`.

The portal UI is now backed by live project, service, and monitoring sources. Remaining work is verification, UI rebaseline, and cleanup.

## Priority

1. Verify the scheduled catalog-sync CronJob stays green.
2. Re-run end-to-end validation and capture fresh dev evidence.
3. Rebaseline UI state against live data.
4. Remove stale/manual registry artifacts.

## Tickets

### P0.1 Mount or fetch GitOps workloads repo for project sync
- **Status:** DONE (2026-03-06)
- **Problem:** GitOps project sync runs in-cluster, but the backend and catalog-sync job do not have access to the `workloads/` repo contents. `/projects` stays empty.
- **Evidence:** `POST /service-registry/sync?source=gitops_apps&env=dev` returns `sourceFailures[0].error = "GitOps workloads repo path does not exist"`.
- **Acceptance Criteria:**
  - Catalog sync job can read GitOps app definitions in-cluster.
  - `POST /service-registry/sync?source=gitops_apps&env=dev` returns `discovered > 0` and no path-missing failure.
  - `GET /projects?env=dev` returns canonical GitOps-backed rows.
  - `GET /projects/diagnostics?env=dev` reports `freshness.state` as `fresh` or `warning`, not `empty`.
- **Implementation Options:**
  - Mount a checked-out `workloads` repo into the pod and set `GITOPS_WORKLOADS_REPO_PATH`.
  - Clone/fetch the repo during sync and set `GITOPS_WORKLOADS_REPO_URL` and revision metadata.
  - Replace filesystem-based discovery with an in-cluster GitOps source if available.
- **Risk:** Medium

### P0.2 Fix in-cluster service registry sync connectivity
- **Status:** DONE (2026-03-06)
- **Problem:** Live service discovery cannot reach its upstreams, so `/services` remains empty and registry freshness is stale.
- **Evidence:** `POST /service-registry/sync?source=cluster_services&env=dev` reports 5 source failures, including Kubernetes API and Argo CD connection errors.
- **Acceptance Criteria:**
  - Cluster sync completes with `sourceFailures = []` in dev.
  - `GET /services?env=dev` returns live cluster-backed rows.
  - `GET /services/{serviceId}?env=dev` returns a real service instead of `404`.
  - `GET /service-registry/diagnostics?env=dev` reports `freshness.state` as `fresh` or `warning`.
- **Likely Work Areas:**
  - Kubernetes API service DNS/access from the backend pod.
  - Service account RBAC for namespace/resource discovery.
  - Argo CD in-cluster URL/auth/TLS wiring.
  - NetworkPolicy rules between `homelab-api`, `argocd`, and core cluster services.
- **Risk:** High

### P0.3 Restore Prometheus reachability from the API pod
- **Status:** DONE (2026-03-06)
- **Problem:** Metrics endpoints fail because Prometheus is unreachable from the backend.
- **Evidence:** `GET /services/homelab-api/metrics/summary?range=24h` and `GET /services/homelab-api/metrics-summary?range=24h` return structured `502` with `provider="prometheus"` and `error="[Errno -2] Name or service not known"`.
- **Root Cause:** Backend defaulted to `prometheus.monitoring.svc.cluster.local`, but this monitoring stack expects `prometheus-operated.monitoring.svc.cluster.local`; `homelab-api` also needs explicit egress to `monitoring` on port `9090`.
- **Acceptance Criteria:**
  - `GET /monitoring/providers/diagnostics` reports Prometheus `status="healthy"` in dev.
  - Metrics summary endpoints return `200` with `providerStatus.status="healthy"`.
  - Health timeline endpoint returns `200` with timeline segments.
- **Likely Work Areas:**
  - Prometheus service name / namespace mismatch.
  - Cluster DNS resolution from `homelab-api`.
  - NetworkPolicy egress from `homelab-api` to monitoring namespace.
- **Risk:** Medium

### P0.4 Restore Alertmanager reachability from the API pod
- **Status:** DONE (2026-03-06)
- **Problem:** Alerts endpoint degrades gracefully, but cannot reach Alertmanager.
- **Evidence:** Earlier `GET /alerts/active?env=dev&serviceId=homelab-api&limit=50` returned `providerStatus.status="unreachable"` and later narrowed to `Connection refused` on `9093`.
- **Root Cause:** Dev `kube-prometheus-stack` needed `alertmanager.enabled=true`, and `homelab-api` also needed explicit egress to the `monitoring` namespace on port `9093`.
- **Acceptance Criteria:**
  - `GET /monitoring/providers/diagnostics` reports Alertmanager `status="healthy"` in dev.
  - `GET /alerts/active` returns a healthy `providerStatus`.
  - `GET /monitoring/incidents` no longer reports Alertmanager unreachable unless there are genuine upstream incidents.
- **Risk:** Medium

### P0.5 Restore Loki reachability from the API pod
- **Status:** DONE (2026-03-06)
- **Problem:** Logs quickview cannot connect to Loki.
- **Evidence:** `GET /services/homelab-api/logs/quickview?...` returns structured `502` with `provider="loki"` and `error="[Errno 111] Connection refused"`.
- **Root Cause:** `homelab-api` runs under default-deny egress and had no allow rule to the `monitoring` namespace on Loki port `3100`.
- **Acceptance Criteria:**
  - `GET /monitoring/providers/diagnostics` reports Loki `status="healthy"` in dev.
  - `GET /services/{serviceId}/logs/quickview` returns bounded log lines with `providerStatus.status="healthy"`.
  - Loki failure mode is removed from portal warnings for healthy clusters.
- **Risk:** Medium

### P1.1 Verify catalog-sync CronJob end-to-end and alert on failure
- **Status:** DONE (2026-03-06)
- **Problem:** The job packaging and image wiring were fixed, but the scheduled sync path still needs a successful end-to-end run in dev.
- **Evidence:** Earlier failures included placeholder images, missing `/app/scripts/sync_catalog_registries.py`, missing GitOps repo access, missing Kubernetes API `6443` egress, and missing Alertmanager `9093` egress. The latest scheduled CronJob run `homelab-api-catalog-sync-29546770` completed successfully on 2026-03-06, and a manual validation run emitted `catalog_sync_source_result` for both `gitops_apps` and `cluster_services` plus `catalog_sync_run_result ... has_failures=False exit_code=0`.
- **Acceptance Criteria:**
  - Latest `homelab-api-catalog-sync` Job completes successfully.
  - Job logs show both `gitops_apps` and `cluster_services` summaries.
  - Failure signals are visible in logs and can be scraped or alerted on.
  - Runbook documents the expected success log shape and triage flow.
- **Risk:** Medium

### P1.2 Rebaseline portal empty-state diagnostics after source recovery
- **Status:** DONE (2026-03-06)
- **Problem:** The frontend currently renders empty datasets correctly, but once sources recover we need to verify that degraded, warning, and fresh states are accurate and no stale fallback assumptions remain.
- **Evidence:** Projects and services diagnostics now report `fresh`; monitoring providers report `healthy`; the frontend now treats `warning` as a first-class source state, surfaces project/service catalog freshness on the platform health page, and only falls back from `/services` to `/projects` when the live services endpoint is actually unavailable rather than when it merely returns empty or errors.
- **Acceptance Criteria:**
  - Projects page shows GitOps-backed rows with accurate freshness state.
  - Services page shows cluster-backed rows with accurate freshness state.
  - Platform health page distinguishes `healthy`, `warning`, `stale`, and provider-unreachable states with no false empties.
  - Cross-links between projects, services, and releases resolve against live data.
- **Risk:** Low

### P1.3 Re-run live validation suite and capture dev evidence
- **Status:** DONE (2026-03-06)
- **Problem:** The validation tooling exists, but current dev failures prevent a clean end-to-end report.
- **Evidence:** `scripts/live_catalog_validation.py` passed against `http://api.dev.homelab.local` on 2026-03-06 and wrote `/tmp/live-catalog-validation-dev-2026-03-06.json`. The resulting report showed `projects.count = 2`, `services.count = 3`, `releases.count = 2`, fresh project/service diagnostics, and healthy Prometheus plus Alertmanager provider status.
- **Acceptance Criteria:**
  - `scripts/live_catalog_validation.py` passes against `http://api.dev.homelab.local`.
  - Validation report includes non-empty `/projects` and `/services`.
  - Validation report confirms monitoring endpoints are healthy or intentionally degraded with known cause.
  - Evidence is captured in a dated report artifact for dev.
- **Risk:** Low

### P1.4 Investigate healthy-but-empty Prometheus metrics coverage
- **Status:** IN PROGRESS (2026-03-06)
- **Problem:** Live metrics validation now succeeds with Prometheus healthy, but the summary response still reports no usable data for all metric fields.
- **Evidence:** `scripts/live_catalog_validation.py` passed on 2026-03-06 with `metrics.provider = "prometheus"`, `metrics.status = "healthy"`, and `metrics.noDataFields = 4` for `serviceId = "homelab-api"`. After the service-registry metadata fix, live summary improved to `noDataFields = 3` with `restartCount` populated. A local follow-up now switches `uptimePct` to deployment-availability metrics and adds explicit UI/runbook messaging that request-level latency/error metrics remain unavailable without service HTTP instrumentation.
- **Acceptance Criteria:**
  - Metrics summary returns at least one populated field for a known live service in dev, or
  - The no-data result is explained and documented as expected for the current metric set/workload instrumentation.
  - Query templates and label selectors are validated against the actual Prometheus series emitted by `homelab-api` and `homelab-web`.
  - Any expected no-data state is surfaced intentionally in runbooks or UI copy instead of being treated as an implicit success.
- **Risk:** Low

### P2.1 Clean up stale manual or test-era registry rows
- **Status:** IN PROGRESS (2026-03-06)
- **Problem:** Historical data includes manual/test artifacts (`Allowed`, `E2E Project`) and previous duplicate-key failures. These can confuse diagnostics during recovery.
- **Evidence:** Postgres logs show duplicate key violations on `uq_service_registry_name_namespace_env` for manual/test rows on 2026-03-05.
- **Progress:** Backend sync now prunes conflicting non-`cluster_services` rows before canonical upsert, and a new Alembic migration deletes existing non-canonical `service_registry` rows during rollout. Live cluster verification is still needed after the migration applies.
- **Acceptance Criteria:**
  - Manual/test-only rows are removed or explicitly marked non-canonical.
  - Registry uniqueness guarantees hold under repeated sync runs.
  - Diagnostics reflect only canonical GitOps and cluster data.
- **Risk:** Low

## Suggested execution order

1. P0.1 GitOps repo access
2. P0.2 Cluster and Argo sync connectivity
3. P0.3, P0.4, P0.5 monitoring provider reachability
4. P1.1 CronJob green run verification
5. P1.2 portal UI rebaseline
6. P1.3 end-to-end validation rerun
7. P2.1 data cleanup
