# Homelab Portal Issues

Last updated: 2026-03-09

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
- **Status:** DONE (2026-03-06)
- **Problem:** Live metrics validation now succeeds with Prometheus healthy, but the summary response still reports no usable data for all metric fields.
- **Evidence:** `scripts/live_catalog_validation.py` first passed on 2026-03-06 with `metrics.provider = "prometheus"` and `metrics.noDataFields = 4` for `serviceId = "homelab-api"`. The gap was closed by adding backend HTTP Prometheus instrumentation, scraping it via a `ServiceMonitor`, switching uptime/timeline availability to deployment metrics, and coercing missing 5xx samples to `0`. Live dev now returns populated metrics summary fields for `homelab-api`: `uptimePct = 99.09722222222223`, `p95LatencyMs = 24.142857142857146`, `errorRatePct = 0`, and `restartCount = 6.222222222222221`, with every `noData` flag set to `false`.
- **Acceptance Criteria:**
  - Metrics summary returns populated `uptimePct`, `restartCount`, `p95LatencyMs`, and `errorRatePct` for a known live service in dev, or
  - Any remaining no-data fields are explicitly documented as expected for the current metric set/workload instrumentation.
  - Query templates and label selectors are validated against the actual Prometheus series emitted by `homelab-api` and `homelab-web`.
  - Any expected no-data state is surfaced intentionally in runbooks or UI copy instead of being treated as an implicit success.
- **Risk:** Low

### P2.1 Clean up stale manual or test-era registry rows
- **Status:** DONE (2026-03-06)
- **Problem:** Historical data includes manual/test artifacts (`Allowed`, `E2E Project`) and previous duplicate-key failures. These can confuse diagnostics during recovery.
- **Evidence:** Postgres logs show duplicate key violations on `uq_service_registry_name_namespace_env` for manual/test rows on 2026-03-05.
- **Evidence (Live):** On 2026-03-06, live `service_registry` in dev contained only three canonical `cluster_services` rows (`homelab-api`, `homelab-web`, `oauth2-proxy`), a direct SQL check returned zero `Allowed` / `E2E Project` / non-`cluster_services` rows, and repeated `POST /service-registry/sync?source=cluster_services&env=dev` runs completed without uniqueness failures.
- **Acceptance Criteria:**
  - Manual/test-only rows are removed or explicitly marked non-canonical.
  - Registry uniqueness guarantees hold under repeated sync runs.
  - Diagnostics reflect only canonical GitOps and cluster data.
- **Risk:** Low

### P2.2 Investigate intermittent Postgres reachability during scheduled catalog syncs
- **Status:** TODO
- **Problem:** Scheduled `homelab-api-catalog-sync` jobs intermittently fail before source sync starts because the initial Postgres connection to `homelab-api-postgres:5432` is refused.
- **Evidence:** On 2026-03-09 at `08:30`, `08:40`, `08:50`, and `09:20` UTC, `catalog_sync_run_error` logged `psycopg.OperationalError: connection failed: connection to server at "10.43.180.172", port 5432 failed: Connection refused`. The failed Job `homelab-api-catalog-sync-29550800` on `node-3` retried twice and both pods exited within 1 second before any source sync began (`"sources": {}`), while later jobs such as `29550820` completed successfully and diagnostics returned to `fresh`.
- **Acceptance Criteria:**
  - Scheduled catalog-sync jobs stop failing with initial `psycopg.connect(...)` connection refusals.
  - A rolling window of recent CronJob runs completes successfully without transient Postgres reachability failures.
  - If the underlying cause is operational rather than app-level, the runbook or tracker records the node/network/service condition and the chosen mitigation.
  - Project and service catalog diagnostics remain `fresh` across repeated scheduled sync intervals.
- **Likely Work Areas:**
  - Transient service/endpoints availability for `homelab-api-postgres`.
  - Node-local networking or kube-proxy behavior on `node-3`.
  - Resource pressure or short-lived readiness/listener gaps on the Postgres pod.
  - CronJob retry timing vs. brief backend unavailability windows.
- **Risk:** Medium

### P1.5 Fix service health timeline window mismatch in the frontend
- **Status:** DONE (2026-03-06)
- **Problem:** The service details UI still offers a `6h` health timeline window, but the backend only accepts `24h` and `7d`.
- **Evidence:** The service-details health timeline adapter now targets `/services/{serviceId}/health/timeline`, the frontend only offers `24h` and `7d`, and live dev timeline responses now render valid segments instead of `422`-driven empty states. A live check on 2026-03-06 returned healthy/degraded/down segments for `GET /services/homelab-api/health/timeline?range=24h&step=5m`.
- **Acceptance Criteria:**
  - Frontend only offers timeline windows the backend accepts, or the backend is extended to support `6h`.
  - Changing the timeline window does not trigger `422` in the browser console.
  - Timeline errors remain visible to the user instead of silently degrading to an empty chart.
- **Risk:** Low

### P1.6 Restore service details deployment/status metadata from live sources
- **Status:** IN PROGRESS (2026-03-07)
- **Problem:** Service details still show placeholder status/version states (`Health: unknown`, `Sync: unknown`, deployed version `N/A`) and deployment history can be empty or unavailable despite live catalogs being healthy.
- **Evidence:** Dev service details for `homelab-api` still render `Deployed Version: N/A`, unknown status badges, and deployment-history empty/unavailable states while other live catalog data is present.
- **Progress:** The backend service detail route now derives `version`, `health`, and `sync` from live release traceability metadata, and a new `/services/{serviceId}/deployments` route returns deployment rows from the same source so the service-details page no longer depends on a missing endpoint. The service-details version card copy was also changed to reflect live metadata instead of claiming the API is still a placeholder. Live dev verification is still needed after rollout.
- **Acceptance Criteria:**
  - Service details status badges are backed by live release/Argo/service data.
  - Deployed version resolves from live release/deployment metadata instead of placeholders.
  - Recent deployment history loads consistently for live services.
- **Risk:** Medium

### P1.7 Restore live logs quickview behavior on service details
- **Status:** IN PROGRESS (2026-03-07)
- **Problem:** Logs panels still degrade to “logs unavailable” or empty-state behavior even when Loki is healthy.
- **Evidence:** The service details page still reports logs as unavailable in dev despite `GET /monitoring/providers/diagnostics` showing Loki healthy.
- **Progress:** The frontend quickview is no longer gated on Grafana URL configuration, so missing Grafana deep-link settings no longer disable Loki quickview outright. The backend logs query path now also accepts and uses live `appLabel` metadata instead of assuming `serviceId == app`, which prevents false empty results for services whose Loki labels differ from canonical service IDs. Local verification passed for `tests/test_logs_quickview.py` and the frontend build; live dev verification is still needed after rollout.
- **Acceptance Criteria:**
  - Service details logs quickview loads bounded lines for a known live service when Loki is healthy.
  - User-visible errors distinguish `no matching log lines` from request/config/provider failures.
  - Console/network errors for logs quickview are eliminated for healthy providers.
- **Risk:** Medium

### P1.8 Reconcile live service count and endpoint visibility in the portal
- **Status:** IN PROGRESS (2026-03-07)
- **Problem:** Live cluster diagnostics report three services, but some portal surfaces still appear to show only two or provide `No public/internal endpoints available` for live services.
- **Evidence:** `/services?env=dev` returns `homelab-api`, `homelab-web`, and `oauth2-proxy`, while the portal UI still appears to expose only two primary services and shows missing-endpoint states for some live details pages.
- **Progress:** The services adapter now merges live `/services` rows with project metadata so public URLs, internal URLs, and last deploy timestamps are preserved without dropping service-only rows such as `oauth2-proxy`. The services page also calls out service-only rows explicitly, and the service details endpoint state now distinguishes `no routed endpoint` from `metadata missing`. Live verification after rollout is still needed.
- **Acceptance Criteria:**
  - Portal list/detail views consistently reflect all live `/services` rows intended for operator visibility.
  - Endpoint rendering distinguishes `no routed endpoint` from `metadata missing`.
  - Any intentional filtering of service-only rows such as `oauth2-proxy` is documented in UI copy or runbooks.
- **Risk:** Low

### P1.9 Restore live commit, image, Argo sync, and deployed metadata on the release dashboard tab
- **Status:** DONE (2026-03-09)
- **Problem:** The release dashboard still shows placeholder values such as `Upstream unknown` or `Unknown` for commit, image, Argo sync, and deployed state even though live release and Argo metadata exist for `homelab-api` and `homelab-web`.
- **Evidence:** On 2026-03-09 the release dashboard tab still showed `Commit = Upstream unknown`, `Image = Upstream unknown`, `Argo Sync = Unknown`, and `Deployed = Upstream unknown` for both `homelab-api` and `homelab-web`, while live `/services/{serviceId}` data already resolved real `version`, `health`, and `sync` values for `homelab-api`. This was fixed by enriching `/releases` with live Deployment/Argo fallback metadata and by letting the dashboard adapter fall back to `argo.revision` for commit display. Live `GET /releases?env=dev&limit=50` now returns real `commitSha`, `imageRef`, `argo.appName`, `argo.syncStatus`, `argo.healthStatus`, and `deployedAt` values for both `homelab-api` and `homelab-web`.
- **Acceptance Criteria:**
  - Release dashboard rows show commit SHA, image reference, Argo sync state, and deployed timestamp/status when live metadata exists.
  - Placeholder copy is only shown when upstream data is genuinely missing, not when adapters fail to map available fields.
  - Dashboard drift and sync indicators stay consistent with the underlying release traceability API.
- **Risk:** Medium

### P1.10 Restore live status and last-deploy fields on the services list
- **Status:** IN PROGRESS (2026-03-09)
- **Problem:** The services list still renders `Health: unknown`, `Sync: unknown`, and `Last Deploy: N/A` for live services even though cluster sync and service details now resolve real metadata.
- **Evidence:** On 2026-03-09 the Services page still showed `Health: unknown`, `Sync: unknown`, and `Last Deploy: N/A` for `homelab-api` and `homelab-web`, while `GET /services/homelab-api?env=dev` returned `health = "healthy"`, `sync = "synced"`, and `version = "sha-d642ac..."`.
- **Progress:** The frontend services adapter now enriches registry rows with live `GET /releases?limit=...` traceability data plus `GET /services/{serviceId}/metrics/summary`, so list rows can pick up real health/sync, last deploy, and uptime values instead of staying at registry defaults or depending on partially populated service-detail responses. Live browser verification after rollout is still needed.
- **Acceptance Criteria:**
  - Services list rows reflect live health/sync metadata instead of default unknown badges when the service detail endpoint has real values.
  - Last deploy is populated from live deployment/release metadata where available.
  - Services list cards no longer show false `No Data` uptime states when live metrics exist for the service.
- **Risk:** Medium

### P1.11 Restore service details overview cards from live metadata
- **Status:** IN PROGRESS (2026-03-09)
- **Problem:** The Service Overview tab still shows `Deployed Version: N/A` and `Service Status: unknown` for some live services even after backend metadata restoration.
- **Evidence:** On 2026-03-09 the `homelab-web` service page still rendered `Deployed Version: N/A` and `Health: unknown / Sync: unknown` in the overview cards, despite the service-details backend work and live service metadata improvements already shipping for `homelab-api`.
- **Progress:** The service-details frontend now uses live `/releases?serviceId=...` traceability rows as a fallback source for version, health/sync, and deployment cards when the direct service-detail payload remains unresolved or deployment history is empty. Live browser verification after rollout is still needed.
- **Acceptance Criteria:**
  - Overview cards on the Service page render live deployed version, health, and sync values for both `homelab-api` and `homelab-web`.
  - Placeholder copy is replaced by accurate live-state messaging or a precise missing-data reason.
  - Frontend identity/adapters use the resolved service detail payload instead of stale fallback values.
- **Risk:** Medium

### P1.12 Restore service-details observability embeds and quick links
- **Status:** IN PROGRESS (2026-03-09)
- **Problem:** The Service page still shows `Grafana unavailable` and unavailable Argo/Grafana quick links, leaving latency/error panels and external navigation degraded even when backend observability data is available.
- **Evidence:** On 2026-03-09 the service page still rendered `Grafana embed URL is not configured`, `Argo CD Application unavailable due to missing monitoring URL configuration`, and `Grafana Dashboard unavailable due to missing monitoring URL configuration`.
- **Progress:** The frontend now derives Argo links from the resolved `argoAppName` instead of the raw service id, and the homelab frontend config now infers the repo's real Argo/Grafana base URLs on homelab hosts so service-details embeds and quick links are not permanently disabled by empty defaults. Live browser verification after rollout is still needed.
- **Acceptance Criteria:**
  - Latency & Error Trends panels render usable embeds or a precise intentional disabled-state tied to missing config.
  - Quick links for Argo CD, Grafana dashboard, and logs resolve correctly when URLs are configured.
  - UI copy distinguishes missing operator configuration from provider/data failures.
- **Risk:** Low

### P1.13 Restore recent deployments panel and deployment history on service details
- **Status:** IN PROGRESS (2026-03-09)
- **Problem:** The Service page still reports `Deployment history unavailable` or empty recent deployments even when live `/services/{serviceId}/deployments` data exists.
- **Evidence:** On 2026-03-09 the Service page still showed `Deployment history unavailable` / `Service endpoint is not available in this backend` for Recent Deployments, despite `GET /services/homelab-api/deployments?env=dev` returning a live deployment row with version and deployed timestamp.
- **Progress:** The frontend deployment-history adapter now falls back to `/releases?serviceId=...` traceability rows when the direct deployments endpoint is unavailable or empty, so both the Recent Deployments panel and the full deployment-history page can still render live version/status/deployed timestamps. Live browser verification after rollout is still needed.
- **Acceptance Criteria:**
  - Recent Deployments on the Service page renders live deployment rows when `/services/{serviceId}/deployments` returns data.
  - “Open full history” routes to a working deployment history experience for the service.
  - Error states distinguish missing backend support from transient request failures.
- **Risk:** Medium

## Suggested execution order

1. P0.1 GitOps repo access
2. P0.2 Cluster and Argo sync connectivity
3. P0.3, P0.4, P0.5 monitoring provider reachability
4. P1.1 CronJob green run verification
5. P1.2 portal UI rebaseline
6. P1.3 end-to-end validation rerun
7. P1.4 metrics coverage
8. P1.5 timeline window contract
9. P1.6, P1.7, P1.8 service details live-data cleanup
10. P1.9 release dashboard traceability cleanup
11. P1.10 services list status/last deploy cleanup
12. P1.11, P1.12, P1.13 service details UI cleanup
13. P2.2 intermittent Postgres reachability during scheduled syncs
