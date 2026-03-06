# Homelab Portal Issues

Last updated: 2026-03-06

## Current dev state

Observed against `http://api.dev.homelab.local` on 2026-03-06:

- API process is healthy: `GET /health` returns `200 {"status":"ok"}`.
- `/projects`, `/services`, `/releases`, and `/release-dashboard` return empty payloads.
- `POST /service-registry/sync?source=gitops_apps&env=dev` reports `GitOps workloads repo path does not exist`.
- `POST /service-registry/sync?source=cluster_services&env=dev` reports upstream connection failures to Kubernetes and Argo CD.
- `GET /health?includeProviders=true` reports all monitoring providers degraded/unreachable.
- Metrics and timeline endpoints return structured `502` responses for Prometheus reachability failures.
- `GET /alerts/active` degrades to `200` with empty alerts and `providerStatus.status="unreachable"`.
- Loki quickview returns structured `502` with `provider="loki"` and `status="unreachable"`.

The portal UI is therefore up, but most live data contracts are still blocked by missing in-cluster source connectivity.

## Priority

1. Restore GitOps project sync in-cluster.
2. Restore live service registry sync from Kubernetes and Argo CD.
3. Restore monitoring provider reachability.
4. Re-run end-to-end validation and remove stale/empty catalog state.

## Tickets

### P0.1 Mount or fetch GitOps workloads repo for project sync
- **Status:** IN PROGRESS (code complete, awaiting cluster validation)
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
- **Status:** IN PROGRESS (RBAC and API-server egress patch staged, awaiting cluster validation)
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
- **Status:** IN PROGRESS (Prometheus service-name and egress patch staged, awaiting cluster validation)
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
- **Status:** IN PROGRESS (dev monitoring stack patch staged, awaiting cluster validation)
- **Problem:** Alerts endpoint degrades gracefully, but cannot reach Alertmanager.
- **Evidence:** `GET /alerts/active?env=dev&serviceId=homelab-api&limit=50` returns `200` with `alerts=[]` and `providerStatus.status="unreachable"`, `error="[Errno -2] Name or service not known"`.
- **Root Cause:** Dev `kube-prometheus-stack` currently sets `alertmanager.enabled=false`, so `alertmanager-operated.monitoring.svc.cluster.local` does not exist.
- **Acceptance Criteria:**
  - `GET /monitoring/providers/diagnostics` reports Alertmanager `status="healthy"` in dev.
  - `GET /alerts/active` returns a healthy `providerStatus`.
  - `GET /monitoring/incidents` no longer reports Alertmanager unreachable unless there are genuine upstream incidents.
- **Risk:** Medium

### P0.5 Restore Loki reachability from the API pod
- **Status:** IN PROGRESS (API-to-monitoring egress patch staged, awaiting cluster validation)
- **Problem:** Logs quickview cannot connect to Loki.
- **Evidence:** `GET /services/homelab-api/logs/quickview?...` returns structured `502` with `provider="loki"` and `error="[Errno 111] Connection refused"`.
- **Root Cause:** `homelab-api` runs under default-deny egress and had no allow rule to the `monitoring` namespace on Loki port `3100`.
- **Acceptance Criteria:**
  - `GET /monitoring/providers/diagnostics` reports Loki `status="healthy"` in dev.
  - `GET /services/{serviceId}/logs/quickview` returns bounded log lines with `providerStatus.status="healthy"`.
  - Loki failure mode is removed from portal warnings for healthy clusters.
- **Risk:** Medium

### P1.1 Verify catalog-sync CronJob end-to-end and alert on failure
- **Status:** TODO
- **Problem:** The job packaging and image wiring were fixed, but the scheduled sync path still needs a successful end-to-end run in dev.
- **Evidence:** Earlier failures included placeholder images and missing `/app/scripts/sync_catalog_registries.py`; this path was fixed on 2026-03-06 but not yet validated as green in-cluster.
- **Acceptance Criteria:**
  - Latest `homelab-api-catalog-sync` Job completes successfully.
  - Job logs show both `gitops_apps` and `cluster_services` summaries.
  - Failure signals are visible in logs and can be scraped or alerted on.
  - Runbook documents the expected success log shape and triage flow.
- **Risk:** Medium

### P1.2 Rebaseline portal empty-state diagnostics after source recovery
- **Status:** TODO
- **Problem:** The frontend currently renders empty datasets correctly, but once sources recover we need to verify that degraded, warning, and fresh states are accurate and no stale fallback assumptions remain.
- **Acceptance Criteria:**
  - Projects page shows GitOps-backed rows with accurate freshness state.
  - Services page shows cluster-backed rows with accurate freshness state.
  - Platform health page distinguishes `healthy`, `warning`, `stale`, and provider-unreachable states with no false empties.
  - Cross-links between projects, services, and releases resolve against live data.
- **Risk:** Low

### P1.3 Re-run live validation suite and capture dev evidence
- **Status:** TODO
- **Problem:** The validation tooling exists, but current dev failures prevent a clean end-to-end report.
- **Acceptance Criteria:**
  - `scripts/live_catalog_validation.py` passes against `http://api.dev.homelab.local`.
  - Validation report includes non-empty `/projects` and `/services`.
  - Validation report confirms monitoring endpoints are healthy or intentionally degraded with known cause.
  - Evidence is captured in a dated report artifact for dev.
- **Risk:** Low

### P2.1 Clean up stale manual or test-era registry rows
- **Status:** TODO
- **Problem:** Historical data includes manual/test artifacts (`Allowed`, `E2E Project`) and previous duplicate-key failures. These can confuse diagnostics during recovery.
- **Evidence:** Postgres logs show duplicate key violations on `uq_service_registry_name_namespace_env` for manual/test rows on 2026-03-05.
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
