# Homelab Pre-Level-3 TODO

Last updated: 2026-03-14

This backlog tracks the work required to move from Phase 5 read-only observability into Portal Level 3.

Source of truth:

- `docs/homelab.md` readiness gates
- `docs/runbooks/portal-level3-readiness-audit.md`

This file has two sections:

- completed pre-Level-3 readiness gates (`P0.x`)
- the remaining Level 3 entry checklist blockers (`L3.x`)

It does not replace the detailed future Phase 6 implementation work in `T6.x`; the `L3.x` tickets below are a thin operator backlog view over the specific `T6.x` items that still block Level 3.

## Exit condition

- Every `P0.x` ticket is `DONE`.
- Every `L3.x` ticket below is `DONE`.
- `docs/runbooks/portal-level3-readiness-audit.md` evaluates to `GO`.
- Dated evidence is captured in `docs/homelab.md` or `docs/homelab-issues.md`.

## Priority

1. Add durable deployment records and lifecycle state.
2. Add automatic rollout verification and deploy locks.
3. Anchor rollback, observability, and timeline UX to real deployment records.

## Tickets

### P0.1 Establish a 14-28 day multi-service stability evidence window
- **Status:** DONE (2026-03-09)
- **Problem:** The Level 3 readiness gate requires at least 14 consecutive days of stable operation for 3 canonical services with no force-sync or manual cluster intervention.
- **Existing Evidence:** `docs/homelab-issues.md` records 3 canonical live services in dev and healthy platform endpoints.
- **Evidence (Resolved):** Per operator confirmation on 2026-03-09, `homelab-api`, `homelab-web`, and `oauth2-proxy` have been deployed and stable for at least 14 days, implying a stable window starting on or before 2026-02-23, with no force-sync or manual cluster intervention required.
- **Acceptance Criteria:**
  - At least 3 canonical services stay healthy and synced for 14-28 consecutive days.
  - No force-sync or manual cluster intervention is required during that window.
  - Weekly or more frequent dated evidence is captured from `/services`, `/releases`, and registry diagnostics.
  - Any incident during the window is either absent or explicitly documented as not requiring manual recovery.
- **Risk:** High
- **Notes:** Keep appending dated evidence during normal operation, but this gate is now considered satisfied.

### P0.2 Capture live proof for immutable CI artifacts
- **Status:** DONE (2026-03-09)
- **Problem:** The CI workflow exists, but the readiness gate requires dated proof that merged commits produce immutable and auditable artifacts in practice, not just by design.
- **Existing Evidence:** `apps/portal/.github/workflows/portal-images.yml` publishes `sha-<commit>` GHCR tags and retains SBOM/provenance output. `docs/architecture/deployment-flow.md` documents the path.
- **Evidence (Resolved):** On 2026-03-09, three recent successful `portal-images.yml` runs on `main` were verified end-to-end against live GitHub Actions, GitOps PRs, retained SBOM artifacts, and authenticated GHCR manifest checks:
  - `cd76e5907910d89a50a9ad93a635b726007fde5e` -> run `#22856331866` -> GitOps PR `#35`
  - `f3e0f15f5678d47f9efe0caf9ab8e2c2eaf67d2d` -> run `#22853134726` -> GitOps PR `#34`
  - `6beb5ac154c3e0df2009eefd904b6f9c33318412` -> run `#22852594704` -> GitOps PR `#33`
  All six GHCR manifest checks (`homelab-api` and `homelab-web` for each SHA) returned `ok`, and SBOM artifacts were retained for each run.
- **Acceptance Criteria:**
  - At least 3 recent merges to `main` are recorded with matching commit SHA, workflow run, published GHCR tag, and GitOps PR URL.
  - Each recorded run retains SBOM or provenance artifacts.
  - The published image tag remains commit-traceable and immutable.
  - Evidence is summarized in a dated note or table that can be reused by the readiness audit.
- **Risk:** Medium
- **Notes:** The evidence table is recorded in `docs/homelab-issues.md`.

### P0.3 Prove the image promotion path with repeated successful runs
- **Status:** DONE (2026-03-09)
- **Problem:** The repo documents the mechanics for dev auto-bumps and gated promotion, but it does not yet prove the promotion path over repeated real runs.
- **Existing Evidence:** `apps/portal/.github/workflows/portal-images.yml` opens dev image bump PRs. `apps/portal/.github/workflows/gated-promotion.yml` supports approved promote/rollback flows. `docs/homelab.md` records one local rollback rehearsal on 2026-03-04.
- **Evidence (Resolved):** On 2026-03-09, recent dev auto-bump runs were reviewed against GitHub Actions, `homelab-workloads` PRs, and Argo app history for `homelab-api-dev` and `homelab-web-dev`. Nine recent merged auto-bump PRs were confirmed end-to-end with Argo deployment timestamps on both apps. PR `#34` (`automation/dev-image-bump-f3e0f15f5678d47f9efe0caf9ab8e2c2eaf67d2d`) was intentionally closed by the operator after forgetting to merge it before a newer push; it was treated as a skipped/superseded promotion rather than a GitOps or Argo failure. Based on operator review, the promotion path is considered proven for readiness purposes.
- **Acceptance Criteria:**
  - One of these paths is proven:
    - More than 10 consecutive successful dev auto-bumps with zero failed syncs, or
    - At least 3 PR-based manual bump or promotion runs tested end-to-end with zero failed syncs.
  - Each recorded run includes workflow URL, PR URL, merge SHA, and Argo outcome.
  - Any no-op or skipped promotion is excluded from the evidence count.
- **Risk:** High
- **Notes:** The detailed run log and operator exception note are recorded in `docs/homelab-issues.md`.

### P0.4 Integrate the chosen secrets strategy into the live GitOps path
- **Status:** DONE (2026-03-09)
- **Problem:** SOPS needed to move from “chosen on paper” to a real Git-to-Argo delivery path.
- **Existing Evidence:** `docs/runbooks/sops-secrets.md` documents `SOPS + age + KSOPS`; `ansible/roles/argocd/tasks/main.yml` patches `argocd-repo-server` for `ksops`; `workloads/apps/homelab-api/envs/dev/postgres-secret-generator.yaml`, `workloads/apps/homelab-api/envs/prod/postgres-secret-generator.yaml`, and `workloads/apps/homelab-web/envs/dev/oauth2-proxy-secret-generator.yaml` wire encrypted secrets into overlays; `workloads/apps/homelab-web/envs/dev/oauth2-proxy-secret.enc.yaml` is present in Git; `docs/homelab-issues.md` records the live repo-server rollout and the post-push Argo validation at revision `5d3095f`.
- **Acceptance Criteria:**
  - SOPS remains the chosen standard or an explicit alternative is adopted and documented.
  - Secrets committed to Git are encrypted at rest.
  - The live delivery path decrypts and applies secrets safely.
  - At least 2 services successfully consume decrypted secrets with no runtime issues.
  - Guardrails still fail if plaintext `Secret` manifests are committed.
- **Risk:** High
- **Notes:** Closed on 2026-03-09 after Argo rendered `Secret` manifests for `homelab-api-postgres` and `oauth2-proxy-secret` from `homelab-workloads` revision `5d3095f`, with both apps remaining `Synced` and `Healthy`.

### P0.5 Run and capture one real observability alert drill
- **Status:** DONE (2026-03-09)
- **Problem:** Prometheus, Loki, and Alertmanager are healthy, but the readiness gate also requires proof that a real alert fired and was confirmed end-to-end.
- **Existing Evidence:** `docs/homelab-issues.md` records healthy Prometheus, Loki, and Alertmanager providers in dev as of 2026-03-06.
- **Evidence (Resolved):** On 2026-03-09, a temporary `PrometheusRule` named `homelab-observability-drill` was applied in namespace `monitoring` with alert `HomelabObservabilityDrill`, `expr: vector(1)`, `for: 1m`, and labels `service=homelab-api`, `env=dev`, `severity=warning`. Prometheus showed the alert firing at `2026-03-09T18:03:27Z`; Alertmanager returned the active alert with `startsAt = 2026-03-09T18:02:31.101Z`; `GET /alerts/active?env=dev&serviceId=homelab-api&limit=50` returned the alert through the portal with healthy Alertmanager provider status; `GET /services/homelab-api/metrics/summary?range=1h` returned healthy Prometheus-backed metrics for the same service and window; and Prometheus pod logs captured the config reload at `2026-03-09T18:01:16Z` when the drill rule landed. After deleting the temporary rule, Prometheus, Alertmanager, and the portal feed all showed the alert cleared by `2026-03-09T18:10:31Z`.
- **Acceptance Criteria:**
  - One real alert is intentionally triggered or captured from a genuine event.
  - The alert is visible through the expected path: Prometheus rule evaluation, Alertmanager, and portal or Grafana surfaces as applicable.
  - Related logs and dashboard views are captured for the same event window.
  - A dated note records what fired, why it fired, and how it was verified.
- **Risk:** Medium
- **Notes:** The full dated evidence note is recorded in `docs/homelab-issues.md`.

### P0.6 Perform one real Git-backed rollback and measure Argo recovery time
- **Status:** DONE (2026-03-09)
- **Problem:** There is a rollback runbook and one local rehearsal, but no captured evidence of a real cluster-applied rollback finishing within the required time.
- **Existing Evidence:** `docs/runbooks/gitops-dev-prod-promotion.md` defines the rollback path. `docs/homelab.md` records a local `N -> N+1 -> N` rehearsal on 2026-03-04.
- **Evidence (Resolved):** On 2026-03-09 at `18:22:38Z`, `homelab-workloads` `main` was at revision `d83af79`, Argo showed both `homelab-api-dev` and `homelab-web-dev` `Synced`/`Healthy`, and both live deployments were on image tag `sha-cd76e5907910d89a50a9ad93a635b726007fde5e`. A real Git-backed rollback was then executed by reverting merge commit `8e08f8121fbcc68434fb9404f03a03660e434e7f` with new commit `b11ba312617de122359916a13a16fde3a3dbee86`, which changed only the four expected dev image patch files and restored both apps to `sha-6beb5ac154c3e0df2009eefd904b6f9c33318412`. The revert was pushed to `homelab-workloads/main` at `18:22:38Z`. Argo first showed both apps reconciled to `b11ba312617de122359916a13a16fde3a3dbee86`, `Synced`, `Healthy`, and serving the restored `sha-6beb5ac...` images at `18:24:54Z`, for a measured rollback recovery time of `2m16s`. `kubectl rollout status` then succeeded for both `deploy/homelab-api` and `deploy/homelab-web`.
- **Acceptance Criteria:**
  - One real rollback is executed through the supported Git-backed path.
  - The rollback has a clear before state, rollback commit or PR, and after state.
  - Argo CD applies the rollback within 5 minutes.
  - The service is verified to be back on the known-good version after reconcile.
  - The timestamps and result are captured in a dated evidence note.
- **Risk:** High
- **Notes:** The dated rollback drill evidence is recorded in `docs/homelab-issues.md`.

## P0 Status

All pre-Level-3 readiness gates are now satisfied. The remaining blockers are the Render-like Level 3 checklist items below.

## Level 3 Checklist Backlog

These tickets map directly to the remaining `FAIL` and `PARTIAL` rows in `docs/homelab.md`.

### L3.1 Add durable deployment records for every GitOps mutation
- **Status:** DONE (2026-03-10)
- **Problem:** Durable deployment persistence is now in place and proven across every supported mutation type. In the current single-cluster safety mode, prod promote rows are intentionally not expected to reach `live` because the prod workloads path is empty by design, but row creation and traceability are still verified.
- **Existing Evidence:** `apps/portal/backend/alembic/versions/20260310_0006_create_deployments_table.py` creates the `deployments` table; `apps/portal/backend/app/deployment_records.py` implements repository reads/writes plus request-key lookups; `apps/portal/backend/app/deployment_reconciler.py` discovers recent GitOps PRs from GitHub and creates or updates deployment rows keyed by `gitops-pr:<pr>:<service>:<env>:<action>`; `apps/portal/backend/app/main.py` serves `/services/{service_id}/deployments`, `/deployments`, and `/deployments/reconcile` from deployment records and clears deployment-history cache after reconcile/write operations; `apps/portal/.github/workflows/portal-images.yml`, `apps/portal/.github/workflows/gated-promotion.yml`, and `apps/portal/.github/workflows/gitops-config-change.yml` all use the same PR-keyed request model. Fresh dated evidence now exists for a real dev deploy row (`#46`, `live` for both `homelab-api` and `homelab-web`), a real dev config-change row (`#45`, `live` for `homelab-web`), a real prod rollback row (`#42`, `failed` after the PR was intentionally closed without merge), and a real prod promote row (`#47`, `promote` rows written for both services). `workloads/environments/prod/workloads/kustomization.yaml` remains intentionally empty and `workloads/environments/prod/workloads-app.yaml` keeps `allowEmpty: true`, so current prod promote rows are accepted as verified row creation/traceability rather than live workload rollout. Operator confirmation on 2026-03-10 states the promote path was exercised successfully before prod workloads were intentionally disabled in single-cluster mode.
- **Acceptance Criteria:**
  - A first-class `deployments` table exists with an Alembic migration.
  - Every deploy, promote, rollback, and config-change workflow writes a deployment row.
  - Read APIs use deployment records as the source of truth instead of inferred release/runtime rows.
  - Records carry at least `serviceId`, `env`, `action`, `status`, `requestedAt`, and Git linkage.
- **Suggested Work:**
  - Preserve the single-cluster safety-mode exception in future audits unless prod workloads are re-enabled.
  - If prod workloads are re-enabled later, capture a new `promote -> live` row to replace the current operator-reviewed exception.
- **Maps To:** `T6.6.1`, `T6.6.2`
- **Risk:** High

### L3.2 Persist deploy lifecycle state (`pending` -> `deploying` -> `live` / `failed`)
- **Status:** DONE (2026-03-10)
- **Problem:** Lifecycle state is now persisted and proven live. The remaining work in this area is tuning and maintenance, not missing core behavior.
- **Existing Evidence:** `apps/portal/backend/app/deployment_reconciler.py` maps GitOps PR state plus live Argo/Kubernetes evidence into persisted `pending`, `deploying`, `live`, and `failed` statuses; `apps/portal/backend/app/main.py` exposes `failureReason`, `startedAt`, `finishedAt`, `deployWindowStart`, and `deployWindowEnd` through deployment responses; `apps/portal/frontend/src/pages/service-deployments-page.tsx` renders lifecycle status, Argo state, Git links, and failure text directly from deployment records. Live dated evidence on 2026-03-10 showed `pending` rollback rows for `#42`, `deploying` deploy rows for `#46`, `live` rows for deploy `#46` and config-change `#45`, and `failed` rows for rollback `#42` plus older deploy failures with operator-visible reasons.
- **Acceptance Criteria:**
  - Deployment records transition through `pending`, `deploying`, `live`, or `failed`.
  - Transition timestamps are stored and queryable.
  - Portal APIs and UI render lifecycle state directly from deployment records.
  - Failure states include enough reason/context for operator triage.
- **Suggested Work:**
  - Keep stale-timeout tuning under review as more real rollout timings accumulate.
  - Preserve dated evidence whenever reconcile rules or lifecycle semantics change.
- **Maps To:** `T6.6.2`, `T6.6.3`, `T6.2.4`
- **Risk:** High

### L3.3 Add automatic post-merge rollout verification
- **Status:** DONE (2026-03-10)
- **Problem:** Automatic rollout verification is now proven for the supported mutation paths in the current topology. Promote verification in single-cluster safety mode is necessarily limited to merged-PR detection and deployment-row progression because prod workloads are intentionally absent.
- **Existing Evidence:** `apps/portal/backend/app/deployment_reconciler.py` polls recent GitOps PRs, links each PR to deterministic deployment request keys, preserves terminal states, and evaluates live Argo app state plus live deployment image refs to mark rows `deploying`, `live`, or `failed`; `apps/portal/backend/app/main.py` starts a background deployment reconciler loop when enabled in-cluster, exposes `/deployments/reconcile` for operator verification, and clears deployment-history cache on reconcile/write; `apps/portal/backend/tests/test_deployment_reconciler.py` covers PR parsing, live-success reconciliation, config-change revision matching, timestamp monotonicity, and terminal-state preservation. Live dated evidence on 2026-03-10 showed deploy `#46` moving through real post-merge verification to `live`, config-change `#45` reaching `live` automatically after merge, rollback `#42` moving from `pending` to `failed` after the PR was closed without merge, and promote `#47` producing real prod `promote` rows for both services after the merged `gated-promotion.yml` run. Because `environments/prod/workloads` is intentionally empty with `allowEmpty: true` in single-cluster safety mode, `#47` is accepted as operator-reviewed verification of the promote path rather than a `live` prod workload rollout.
- **Acceptance Criteria:**
  - A watcher links each merged Git mutation to exactly one deployment record.
  - Argo sync result and Kubernetes rollout outcome update that record automatically.
  - Failed or stalled rollouts are marked `failed` with operator-visible reason text.
  - Verification is automatic for deploy, promote, rollback, and config-change flows.
- **Suggested Work:**
  - Keep recording dated live evidence whenever reconciler heuristics or timeout logic change.
  - If prod workloads are re-enabled later, capture a fresh `promote -> live` example to replace the current single-cluster exception.
- **Maps To:** `T6.6.3`
- **Risk:** High

### L3.4 Enforce deploy locks per `serviceId+env`
- **Status:** DONE (2026-03-11)
- **Problem:** Portal-driven mutations could overlap because only the GitHub prod promotion workflow had workflow-level concurrency and there was no backend lock manager.
- **Existing Evidence:** `apps/portal/backend/alembic/versions/20260310_0007_create_deployment_locks_table.py` adds persisted `deployment_locks`; `apps/portal/backend/app/deployment_locks.py`, `apps/portal/backend/app/main.py`, and `apps/portal/backend/app/deployment_reconciler.py` now enforce lock acquire/release plus stale cleanup; and a dated live drill on 2026-03-11 confirmed `deploymentLock` visibility on `GET /services/homelab-api?env=dev`, `409 Conflict` on an overlapping `POST /deployments`, and automatic lock release after the terminal `failed` write for the same `requestKey`.
- **Acceptance Criteria:**
  - A deploy lock exists for each in-flight mutation keyed by `serviceId+env`.
  - Conflicting mutations are rejected or queued with a clear operator-facing response.
  - Locks are released on success, failure, or stale-timeout cleanup.
  - UI disables or warns on overlapping actions for the same service/environment.
- **Suggested Work:**
  - Keep recording dated evidence when stale-timeout or lock-release logic changes.
  - If the portal grows new mutation entrypoints, verify they reuse the same `serviceId+env` lock path.
- **Maps To:** `T6.6.4`, `T6.2.4`
- **Risk:** High

### L3.5 Enforce canonical service identity across portal, Argo, Kubernetes, and monitoring
- **Status:** DONE (2026-03-11)
- **Problem:** The identity contract now has both CI and runtime enforcement. The remaining work in this area is maintenance, not missing enforcement paths.
- **Existing Evidence:** `docs/contracts/service-identity.md` and `docs/runbooks/cluster-identity-normalization.md` define the model; `apps/portal/backend/app/service_identity.py` centralizes canonical `serviceId` rules; `apps/portal/backend/app/service_identity_validation.py` emits per-service drift diagnostics through `/service-registry/diagnostics`; `apps/portal/backend/scripts/live_catalog_validation.py` fails on `identityDrift`; and `workloads/scripts/check-service-identity-contract.sh` runs from `validate-gitops-environment-contract.yml` on relevant repo changes. A dated live pass on 2026-03-11 returned `identityDrift.driftCount = 0`, `okCount = 3`, a passing `live_catalog_validation.py` report, and `HTTP 422` for a non-canonical deployment write using `serviceId = "Homelab API"`.
- **Acceptance Criteria:**
  - CI or runtime validation fails when service identity drifts across GitOps paths, Argo app names, Kubernetes labels, Prometheus selectors, Loki labels, and release joins.
  - Portal write workflows use canonical service identity as the only mutation key.
  - Drift diagnostics are operator-visible and actionable.
  - At least one automated validation path runs on every relevant repo change.
- **Suggested Work:**
  - Preserve dated runtime evidence whenever the identity contract or diagnostics rules change.
  - If live monitoring labels ever diverge from `service_registry`, extend diagnostics to record the exact Loki/Prometheus mismatch payload.
- **Maps To:** `T6.4.4`, `T6.6.5`
- **Risk:** Medium

### L3.6 Make portal rollback a first-class, traceable workflow
- **Status:** DONE (2026-03-11)
- **Problem:** Portal rollback is now first-class and traceable. The remaining work in this area is maintenance, not missing rollback workflow coverage.
- **Existing Evidence:** `apps/portal/backend/app/github_workflows.py` dispatches the existing `gated-promotion.yml` workflow in `rollback` mode; `apps/portal/backend/app/main.py` exposes `POST /rollbacks`; `apps/portal/.github/workflows/gated-promotion.yml` records `operator_reason` into rollback PR bodies plus deployment-record writes; `apps/portal/backend/app/deployment_reconciler.py` parses `- Reason:` lines from rollback PR bodies so operator-provided reason text survives reconcile/backfill; `apps/portal/frontend/src/pages/service-details-page.tsx` renders a rollback request panel for `homelab-api` and `homelab-web`; and dated live evidence on 2026-03-11 proved the full path through portal request -> GitHub Actions run `#22953368415` -> workloads PR `#53` -> real `action=rollback` deployment rows with operator reason plus compare/PR linkage -> reconciled `failed` outcome after intentionally closing the PR without merge. The temporary cluster-only token hotfix used for that first drill was then replaced by a GitOps-managed secret, `homelab-api-github-actions`, at workloads revision `9852145`, and a follow-up `POST /rollbacks` returned `202 Accepted` using the Argo-managed token path.
- **Acceptance Criteria:**
  - The portal can initiate rollback through a supported Git-backed path.
  - Every rollback writes a deployment record with `action=rollback`.
  - Rollback records store reason, compare or PR link, and rollout result.
  - The rollback remains visible in service deployment history and operator audit views.
- **Suggested Work:**
  - Preserve dated evidence whenever rollback request payloads, compare-link generation, or token wiring changes.
  - If prod workloads are re-enabled later, capture a fresh portal rollback that reaches `live` instead of the current safe closed-PR failure drill.
- **Maps To:** `T6.2.3`, `T6.2.4`, `T6.6.2`
- **Risk:** High

### L3.7 Anchor logs and metrics to deploy windows
- **Status:** DONE (2026-03-11)
- **Problem:** Deploy-window observability is now anchored to persisted deployment records and explicit deploy windows. The remaining work in this area is maintenance, not missing deploy-window linkage.
- **Existing Evidence:** `apps/portal/backend/app/main.py` now exposes `/services/{service_id}/observability/window` for deployment-scoped metrics, health timeline, and logs queries by `deploymentId` or explicit `windowStart/windowEnd`; `apps/portal/frontend/src/pages/service-deployments-page.tsx` renders a deployment drilldown panel anchored to the selected deployment record; deployment-record metric snapshots use `deployWindowStart` / `deployWindowEnd` instead of a synthetic `deployedAt + 1h` window; and dated live evidence on 2026-03-11 showed a real `homelab-api` deploy record (`deploymentId = 043ff7f3-252c-48d5-baa7-5650e7e590ea`, PR `#54`) returning `context.evidenceStatus = "resolved"`, `metricStatus = "ok"`, `timelineStatus = "ok"`, and `logsStatus = "no_data"` through both the `deploymentId` path and the explicit `windowStart/windowEnd` path.
- **Acceptance Criteria:**
  - Logs, metrics, and health timeline views can be queried by deployment ID or deployment window.
  - Deployment detail surfaces show before/after or in-window observability tied to the actual deployment record.
  - Missing telemetry is distinguishable from query failure or absent deployment evidence.
  - Observability windows work for deploy, promote, and rollback records.
- **Suggested Work:**
  - Preserve dated evidence whenever deployment-window query semantics or provider fallback behavior changes.
  - If prod workloads are re-enabled later, capture one fresh promote/rollback drill through the same endpoint for completeness.
- **Maps To:** `T6.7.1`, `T6.7.2`, `T6.7.3`
- **Risk:** Medium

### L3.8 Rebuild deployment history timeline with reason and changelog context
- **Status:** DONE (2026-03-11)
- **Problem:** The deployment timeline now comes from deployment records and exposes the full mixed-action history with explicit filtering. The remaining work in this area is maintenance, not missing timeline structure.
- **Existing Evidence:** `apps/portal/frontend/src/lib/adapters/deployments.ts` preserves deployment-record action, actor, reason, failure text, compare links, and PR metadata; `apps/portal/frontend/src/pages/service-deployments-page.tsx` renders requested/completed timestamps, action, version, lifecycle state, Argo sync/health, PR links, compare links, and deploy reason from deployment records rather than only showing a metrics-only table; live deployment history already includes real `deploy` (`#46`), `promote` (`#47`), `config-change` (`#45`), and `rollback` (`#42`) rows; and on 2026-03-11 the page gained explicit action and status filters on top of the existing service/env scope and sorting modes, with a clean production frontend build.
- **Acceptance Criteria:**
  - Deployment timeline rows come from deployment records, not inferred runtime data.
  - Each row shows action, reason, actor, result, compare link, and changelog context where available.
  - Rollback and config-change rows are visible alongside deploy/promote rows.
  - Timeline filters and sorting work by service, environment, action, and status.
- **Suggested Work:**
  - Preserve dated evidence if timeline grouping, filtering semantics, or compare-link rendering changes.
- **Maps To:** `T6.2.4`, `T6.2.5`, `T6.7.2`
- **Risk:** Medium

## After these are done

Once every `L3.x` ticket above is complete, rerun `docs/runbooks/portal-level3-readiness-audit.md`. Level 3 is ready to start only when the readiness gates remain green and the Render-like checklist has no blocking `FAIL` rows.

---

## Feature Backlog

Post-Level-3 feature expansion tickets. These do not block Level 3 entry; they extend the scaffold and portal capability once the core platform is stable.

### T6.4.5 Scaffold template: database add-on (PostgreSQL / MySQL)
- **Status:** DONE (2026-03-14)
- **Description:** Extend the scaffold generator (both `workloads/scripts/scaffold-service.py` and the portal wizard `POST /scaffold/*`) to accept an optional `--add-on database` flag with a `--db-engine postgres|mysql` choice. When selected, the scaffold generates additional manifests alongside the base app:
  - `StatefulSet` for the database with a named `volumeClaimTemplate` (PVC).
  - `Service` (ClusterIP) for in-cluster connectivity.
  - `NetworkPolicy` allowing only the app pods to reach the DB on the engine's port.
  - A SOPS-encrypted `Secret` stub (`<service>-db-credentials`) with placeholder values and a comment pointing to the SOPS runbook.
  - For PostgreSQL: a `migration-job.yaml` (Argo CD Sync hook) modelled on the `homelab-api` pattern — waits for DB readiness, runs `alembic upgrade head` (or a configurable command), retries on failure.
  - For MySQL: equivalent init-job using `mysql -e "SOURCE /init.sql"` pattern; no Alembic dependency.
  - `catalog-service.yaml` observability mode stays `app-native` (unchanged).
- **Acceptance Criteria:**
  - ✓ `--add-on database --db-engine postgres` generates all manifests above; kustomize renders without errors.
  - ✓ `--add-on database --db-engine mysql` generates MySQL equivalents; kustomize renders without errors.
  - ✓ Smoke test extended with a postgres-add-on variant that checks the extra file count and renders both overlays.
  - ⊘ Portal wizard gains a "Database add-on" toggle in Step 2 (Template) with engine selector; preview step shows the extra files. (moved to T6.4.5.1)
  - ✓ `validate-services-catalog.py` passes unchanged (no new catalog fields required).
  - ✓ No plaintext credentials committed; guardrails still catch unencrypted `Secret` kind.
- **Dependencies:** T6.4.2, T6.4.3, T6.4.4
- **Complexity:** M
- **Risk:** Low
- **Notes:** Follow the `homelab-api` Postgres pattern as the reference implementation. MySQL is a parallel path — same structure, different image and port. Keep migration job optional behind a `--migration-command` flag so static sites scaffolded with this add-on don't get an unwanted job. CLI implementation complete and tested 2026-03-14; portal wizard UI deferred to T6.4.5.1.

### T6.4.5.1 Portal wizard: database add-on UI
- **Status:** DONE (2026-03-14)
- **Description:** Add portal wizard UI support for the database add-on feature completed in T6.4.5. Extend the `POST /scaffold/*` endpoint and the frontend wizard component to allow operators to select a database engine (PostgreSQL or MySQL) when creating a new service. The wizard should display database addon files in the preview step and pass the database engine choice to the backend scaffold generator.
- **Acceptance Criteria:**
  - ✅ Portal wizard Step 2 (Template selection) shows a "Database add-on" checkbox or toggle below the template choice.
  - ✅ When "Database add-on" is checked, an engine dropdown appears with options: `postgres` (default) and `mysql`.
  - ✅ Optionally, a "Migration command" text input allows custom migration/init commands (default: `alembic upgrade head` for postgres, empty for mysql).
  - ✅ Preview step lists all generated database addon files: StatefulSet, Service, NetworkPolicy, Secret, and migration/init job.
  - ✅ `POST /scaffold-submit` backend endpoint accepts `addon_engine` and `addon_migration_command` query parameters and passes them to `scaffold-service.py`.
  - ✅ Validation ensures addon engine is only set when addon_database is enabled.
  - ✅ UI disables custom migration input for static-nginx template (no app container to run migrations) if database addon is selected.
- **Dependencies:** T6.4.5, T6.1.3 (existing scaffold submit endpoint)
- **Complexity:** S
- **Risk:** Low
- **Notes:** The backend scaffold generator and CLI are complete (T6.4.5); this ticket was pure UI integration. Reused existing style and patterns from the template and observability-mode selectors. Implementation completed end-to-end with all tests passing (postgres, mysql, baseline variants).

---

### T6.4.6 Scaffold template: WordPress bundle
- **Status:** TODO
- **Description:** Add a `wordpress` template to the scaffold generator and portal wizard. WordPress is a pre-built image (no custom Dockerfile / CI workflow), so the scaffold output differs from the `python-fastapi` and `static-nginx` paths:
  - **No** application repo scaffold and no `.github/workflows/build-*.yml` — the wizard skips the image-repo field or marks it as read-only (`wordpress:latest` or a pinned tag).
  - GitOps manifests:
    - `Deployment` for the WordPress container with `WORDPRESS_DB_*` env vars sourced from a Secret.
    - `PersistentVolumeClaim` for `/var/www/html/wp-content` (uploaded media and plugins).
    - `Service` (ClusterIP).
    - `Ingress` (same Traefik pattern as other templates).
    - MySQL `StatefulSet` + `Service` + `NetworkPolicy` (reuse the DB add-on manifests from T6.4.5).
    - SOPS-encrypted `Secret` stub for `WORDPRESS_DB_PASSWORD` / `MYSQL_ROOT_PASSWORD`.
    - `NetworkPolicy` allowing only the WordPress pod to reach MySQL.
  - `observability.mode: ingress-derived` in `services.yaml` (no `/metrics` endpoint).
  - No `ServiceMonitor` generated.
  - Prod overlay: same single-cluster safety-mode comment as other templates.
- **Acceptance Criteria:**
  - `scaffold-service.py --template wordpress` generates the full manifest set; kustomize renders both overlays without errors.
  - Portal wizard shows `WordPress` as a template option in Step 2; image-repo field is hidden or pre-filled.
  - Smoke test extended with a WordPress variant checking file count (should be fewer than python-fastapi since no servicemonitor/CI workflow).
  - Catalog entry uses `mode: ingress-derived`.
  - Guardrails still catch any unencrypted `Secret` kind in the generated output.
- **Dependencies:** T6.4.2, T6.4.3, T6.4.5 (MySQL add-on manifests)
- **Complexity:** M
- **Risk:** Low
- **Notes:** WordPress upgrades and plugin management are out of scope for the scaffold — the generated manifests are a starting point only. Document the SOPS secret rotation steps in a short runbook section alongside the generated README.

---

### T6.4.7 Public hostname management — service card editor and wizard field
- **Status:** TODO
- **Description:** Allow operators to view and update the public hostname for a service (e.g. `portal.wlodzimierrr.co.uk`) both when registering a new service via the wizard and from an existing service card in the portal. The hostname maps to the Ingress `host` field in the GitOps overlay and is the externally routable DNS name.

  **New service wizard (Step 3 — Configuration):**
  - Add `Public hostname` field (optional). Default pre-filled as `<name>.<base-domain>` derived from a portal config env var `PUBLIC_BASE_DOMAIN`.
  - Separate from `devHost` / `prodHost` — this field sets the production-facing external name; `devHost` remains the internal `.homelab.local` name.
  - Value written into `apps/<name>/envs/prod/patch-ingress.yaml` as the prod Ingress host.
  - Store alongside the catalog entry in `services.yaml` under `envs[prod].public_host`.

  **Service card editor (existing service):**
  - Add a `Public hostname` row to the service settings/details panel (read + edit modes).
  - Edit mode shows an input field pre-filled from `services.yaml` `envs[prod].public_host`.
  - On save: backend creates a GitOps PR that updates `apps/<service>/envs/prod/patch-ingress.yaml` host field and `services.yaml` `envs[prod].public_host` in a single commit; returns PR URL shown inline.
  - Requires `require_admin` auth (same as scaffold submit).
  - Dev hostname remains editable separately or is out of scope for this ticket (internal only).

- **Acceptance Criteria:**
  - Wizard Step 3 shows `Public hostname` field; generated `patch-ingress.yaml` in the prod overlay contains the supplied value.
  - `services.yaml` written by wizard includes `envs[prod].public_host`.
  - `validate-services-catalog.py` accepts the new optional `public_host` field (string, non-empty if present).
  - Service card reads and displays `public_host` from catalog data.
  - Edit + save flow opens a PR with only the two changed files; PR URL returned to UI.
  - No-op save (hostname unchanged) is detected backend-side and returns a `204` without opening a PR.
  - Smoke test updated to verify `public_host` in the generated `services.yaml` when `--prod-host` is supplied.
- **Dependencies:** T6.4.3, T6.4.4, T6.1.4 (GitOps PR flow)
- **Complexity:** M
- **Risk:** Low
- **Notes:** DNS record creation is out of scope — the ticket only manages the Ingress host. Add a note in the PR body reminding the operator to point the DNS record at the cluster ingress IP. `PUBLIC_BASE_DOMAIN` should default to `homelab.local` when not set so the wizard works without configuration in dev.
