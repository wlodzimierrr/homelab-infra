# Homelab Portal Issues

Last updated: 2026-03-11

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

The portal UI is now backed by live project, service, and monitoring sources. The remaining open work is limited to Grafana deep-link polish and an intermittent Postgres connectivity issue on scheduled catalog syncs.

## Priority

1. Verify the scheduled catalog-sync CronJob stays green.
2. Re-run end-to-end validation and capture fresh dev evidence.
3. Rebaseline UI state against live data.
4. Remove stale/manual registry artifacts.

## Readiness evidence

### 2026-03-09 Immutable CI artifact verification

Captured against `wlodzimierrr/homelab-portal` and `wlodzimierrr/homelab-workloads` on 2026-03-09:

| Date | Commit SHA | Workflow run | GitOps PR | GHCR tags | Retained evidence |
| --- | --- | --- | --- | --- | --- |
| 2026-03-09T13:42:56Z | `cd76e5907910d89a50a9ad93a635b726007fde5e` | `#22856331866` | `#35` | `homelab-api:sha-cd76e5907910d89a50a9ad93a635b726007fde5e` = `ok`; `homelab-web:sha-cd76e5907910d89a50a9ad93a635b726007fde5e` = `ok` | Workflow artifacts included `sbom-backend-cd76e5907910d89a50a9ad93a635b726007fde5e` and `sbom-frontend-cd76e5907910d89a50a9ad93a635b726007fde5e`; backend SBOM download was manually verified. |
| 2026-03-09T12:19:59Z | `f3e0f15f5678d47f9efe0caf9ab8e2c2eaf67d2d` | `#22853134726` | `#34` | `homelab-api:sha-f3e0f15f5678d47f9efe0caf9ab8e2c2eaf67d2d` = `ok`; `homelab-web:sha-f3e0f15f5678d47f9efe0caf9ab8e2c2eaf67d2d` = `ok` | Workflow artifacts included `sbom-backend-f3e0f15f5678d47f9efe0caf9ab8e2c2eaf67d2d` and `sbom-frontend-f3e0f15f5678d47f9efe0caf9ab8e2c2eaf67d2d`. |
| 2026-03-09T12:04:47Z | `6beb5ac154c3e0df2009eefd904b6f9c33318412` | `#22852594704` | `#33` | `homelab-api:sha-6beb5ac154c3e0df2009eefd904b6f9c33318412` = `ok`; `homelab-web:sha-6beb5ac154c3e0df2009eefd904b6f9c33318412` = `ok` | Workflow artifacts included `sbom-backend-6beb5ac154c3e0df2009eefd904b6f9c33318412` and `sbom-frontend-6beb5ac154c3e0df2009eefd904b6f9c33318412`. |

Result: the immutable-artifact readiness gate is satisfied for the current audit because recent merged commits can be traced from source commit -> successful image workflow run -> immutable `sha-<commit>` GHCR tags -> GitOps PR, with retained SBOM artifacts.

### 2026-03-09 Dev auto-promotion verification

Captured against `wlodzimierrr/homelab-portal`, `wlodzimierrr/homelab-workloads`, and Argo CD on 2026-03-09:

- Recent `portal-images.yml` runs were correlated with matching `homelab-workloads` PRs and Argo deployment history for both `homelab-api-dev` and `homelab-web-dev`.
- Nine recent merged auto-bump PRs were confirmed end-to-end with retained Argo deployment timestamps on both dev apps:
  - `#35`, `#33`, `#32`, `#31`, `#30`, `#29`, `#28`, `#27`, `#26`
- PR `#34` (`automation/dev-image-bump-f3e0f15f5678d47f9efe0caf9ab8e2c2eaf67d2d`) was closed without merge because the operator forgot to merge it before pushing again. This was reviewed and accepted as a skipped/superseded manual action, not as a GitOps/Argo sync failure.
- Older merged PRs exist as additional evidence, but some detailed per-app Argo history entries are no longer retained for those revisions.

Result: the promotion-readiness gate is accepted for the current audit based on repeated successful dev auto-bumps, live Argo evidence for recent merged runs, and explicit operator confirmation that the only recent interruption was a manual superseded PR rather than a platform fault.

### 2026-03-09 SOPS/KSOPS GitOps integration

Captured against this repo and the live cluster on 2026-03-09:

- `ansible/roles/argocd/tasks/main.yml` now patches `argocd-repo-server` to mount `ksops`, consume the `argocd-sops-age` Secret, and enable plugin-aware Kustomize builds.
- `workloads/apps/homelab-api/envs/dev` and `workloads/apps/homelab-api/envs/prod` now use `postgres-secret-generator.yaml`.
- `workloads/apps/homelab-web/envs/dev` now uses `oauth2-proxy-secret-generator.yaml`.
- The live `oauth2-proxy-secret` was exported from the cluster, stripped to stable fields, and re-encrypted into `workloads/apps/homelab-web/envs/dev/oauth2-proxy-secret.enc.yaml`.
- Local validation passed with a temporary `ksops` binary: `check-secrets-guardrails.sh`, `check-environment-contract.sh`, `validate-sso-integration.sh`, and `smoke-test-scaffold-generator.sh` all succeeded.
- Live Argo repo-server validation passed:
  - `argocd-sops-age` Secret created in namespace `argocd`
  - `argocd-cm.data["kustomize.buildOptions"] = "--enable-alpha-plugins --enable-exec"`
  - `argocd-repo-server` rolled out successfully with `busybox:1.36.1` init container, mounted `/usr/local/bin/ksops`, and mounted `/home/argocd/.config/sops/age/keys.txt`

Post-push validation captured on 2026-03-09:

- `homelab-workloads` advanced to revision `5d3095f` in both `homelab-api-dev` and `homelab-web-dev`.
- `argocd app manifests homelab-api-dev --grpc-web` rendered a real `kind: Secret` object named `homelab-api-postgres`.
- `argocd app manifests homelab-web-dev --grpc-web` rendered a real `kind: Secret` object named `oauth2-proxy-secret`.
- Both Applications remained `Synced` and `Healthy`.
- `kubectl -n homelab-api rollout status deploy/homelab-api --timeout=300s` succeeded.
- `kubectl -n homelab-web rollout status deploy/oauth2-proxy --timeout=300s` succeeded.

Result: the secrets-readiness gate is satisfied. Git-managed encrypted secrets are now decrypted and applied through the live Argo CD delivery path for at least two services with no runtime issues.

### 2026-03-09 Observability alert drill

Captured against the live monitoring stack, the portal API, and the cluster on 2026-03-09:

- A temporary `PrometheusRule` named `homelab-observability-drill` was applied in namespace `monitoring` with:
  - `alert: HomelabObservabilityDrill`
  - `expr: vector(1)`
  - `for: 1m`
  - labels `service=homelab-api`, `env=dev`, `severity=warning`, `drill=readiness`
- The rule was created at `2026-03-09T18:00:36Z`.
- Prometheus confirmed the alert as `firing` at `2026-03-09T18:03:27Z` via:
  - `ALERTS{alertname="HomelabObservabilityDrill",alertstate="firing"}`
- Alertmanager returned the active alert with:
  - `startsAt = 2026-03-09T18:02:31.101Z`
  - `status.state = "active"`
  - labels including `service = "homelab-api"` and `env = "dev"`
- The portal alert surface returned the same active alert at `2026-03-09T18:04:38Z` through:
  - `GET /alerts/active?env=dev&serviceId=homelab-api&limit=50`
  - healthy `providerStatus.provider = "alertmanager"`
- The portal metrics surface returned healthy Prometheus-backed service metrics in the same drill window at `2026-03-09T18:04:38Z` through:
  - `GET /services/homelab-api/metrics/summary?range=1h`
  - `uptimePct = 100`
  - `p95LatencyMs = 24.181818181818183`
  - `errorRatePct = 0`
  - `restartCount = 0`
- Related logs for the same event window were captured directly from the Prometheus pod:
  - `2026-03-09T18:01:16.809Z Loading configuration file`
  - `2026-03-09T18:01:16.850Z Completed loading of configuration file`
  These log lines bracket the rule landing and reload before the alert entered `firing`.
- Cleanup was confirmed:
  - the temporary `PrometheusRule` was deleted
  - by `2026-03-09T18:10:31Z`, Prometheus returned no matching `ALERTS`, Alertmanager returned `[]`, and `GET /alerts/active?env=dev&serviceId=homelab-api&limit=50` returned `total = 0`

Result: the observability-readiness gate is satisfied. A real alert was intentionally triggered, observed through Prometheus, Alertmanager, and the portal alert surface, correlated with live metrics and logs for the same service/window, and then cleared cleanly.

### 2026-03-09 Git-backed rollback drill

Captured against `homelab-workloads`, Argo CD, and the live cluster on 2026-03-09:

- Before rollback at `2026-03-09T18:21:39Z`:
  - `homelab-workloads/main` was at revision `d83af799f4d6f7ed9f9bede5d44193a4cf6584d9`
  - `homelab-api-dev` and `homelab-web-dev` were both `Synced` and `Healthy`
  - both live deployments were on image tag `sha-cd76e5907910d89a50a9ad93a635b726007fde5e`
- A real Git-backed rollback was executed at `2026-03-09T18:22:38Z` by reverting merge commit `8e08f8121fbcc68434fb9404f03a03660e434e7f` in `homelab-workloads`.
- The rollback commit was:
  - `b11ba312617de122359916a13a16fde3a3dbee86`
  - message: `Revert "Merge pull request #35 from wlodzimierrr/automation/dev-image-bump-cd76e5907910d89a50a9ad93a635b726007fde5e"`
- The revert changed only the four expected dev image patch files:
  - `apps/homelab-api/envs/dev/patch-deployment.yaml`
  - `apps/homelab-api/envs/dev/patch-migration-job.yaml`
  - `apps/homelab-api/envs/dev/patch-catalog-sync-cronjob.yaml`
  - `apps/homelab-web/envs/dev/patch-deployment.yaml`
- Those changes restored both apps from `sha-cd76e5907910d89a50a9ad93a635b726007fde5e` back to the previous known-good `sha-6beb5ac154c3e0df2009eefd904b6f9c33318412`.
- The revert was pushed to `homelab-workloads/main` at `2026-03-09T18:22:38Z`.
- Argo reconciliation evidence:
  - both apps remained at old revision `d83af79...` through `18:24:40Z`
  - at `2026-03-09T18:24:54Z`, both `homelab-api-dev` and `homelab-web-dev` reported revision `b11ba312617de122359916a13a16fde3a3dbee86`
  - at that same timestamp, both apps were `Synced` and `Healthy`
  - at that same timestamp, both live deployment images had changed to `sha-6beb5ac154c3e0df2009eefd904b6f9c33318412`
- Measured rollback recovery time:
  - push time: `2026-03-09T18:22:38Z`
  - Argo/livedeploy reconcile confirmation: `2026-03-09T18:24:54Z`
  - total: `2m16s`
- Post-rollback verification succeeded:
  - `kubectl -n homelab-api rollout status deploy/homelab-api --timeout=300s`
  - `kubectl -n homelab-web rollout status deploy/homelab-web --timeout=300s`

Result: the rollback-readiness gate is satisfied. A real Git-backed rollback was executed, Argo applied it in under 5 minutes, and both dev services were verified on the known-good version after reconcile.

### 2026-03-10 Deployment-record and rollout-verification evidence

Captured against `http://api.dev.homelab.local`, `wlodzimierrr/homelab-portal`, and `wlodzimierrr/homelab-workloads` on 2026-03-10:

- The deployment-record API and live reconciler were exercised through real GitOps mutations after the deployment-record backend and reconciler fixes were deployed.
- Fresh canonical evidence set:
  - Deploy `#46` (`automation/dev-image-bump-04aea0869d105ee2332bf7e2ceb668ac4251c675`) reached `live` for both `homelab-api` and `homelab-web`.
    - `requestedAt = 2026-03-10T17:10:11+00:00`
    - `startedAt = 2026-03-10T17:12:55+00:00`
    - `finishedAt = 2026-03-10T17:14:31.375670+00:00`
  - Config-change `#45` (`automation/dev-config-change-homelab-web-replicas-1-22914424849`) reached `live` for `homelab-web`.
    - `requestedAt = 2026-03-10T17:01:47+00:00`
    - `startedAt = 2026-03-10T17:03:28+00:00`
    - `finishedAt = 2026-03-10T17:13:00.758174+00:00`
  - Rollback `#42` (`automation/prod-rollback-image-update-22912496498`) created real prod deployment rows for both `homelab-api` and `homelab-web`, was observed as `pending`, and then was automatically marked `failed` after the PR was intentionally closed without merge.
    - `requestedAt = 2026-03-10T16:22:12+00:00`
    - `finishedAt = 2026-03-10T16:37:20.793463+00:00`
    - `failureReason = "GitOps pull request was closed without merge."`
- Lifecycle proof captured on 2026-03-10:
  - `pending`: rollback `#42` before PR close
  - `deploying`: deploy `#46` after merge and before rollout completion
  - `live`: deploy `#46` and config-change `#45`
  - `failed`: rollback `#42`, plus earlier failed deploy rows with operator-visible reason text
- Reconciler verification was confirmed through the live deployment-record API and `/deployments/reconcile` once the route-order, terminal-state, timestamp, and cache-invalidation fixes were deployed.
- Earlier rows created during reconciler-fix iterations (`#39`, `#41`, `#44`) should not be treated as the canonical proof set for Level 3 readiness because they were captured while verification and cache logic were still being corrected.
- Promote evidence captured on 2026-03-10:
  - `Gated GitOps Promotion` workflow run `#22915432416` succeeded.
  - `homelab-workloads` PR `#47` (`automation/prod-promote-image-update-22915432416`) merged at `2026-03-10T17:27:22Z`.
  - The deployment-record API returned real `action = "promote"` rows for both `homelab-api` and `homelab-web` in `prod`, with `gitPrNumber = 47`, `gitRef = "automation/prod-promote-image-update-22915432416"`, `deployReason = "GitOps promote via PR #47"`, and version `sha-04aea0869d105ee2332bf7e2ceb668ac4251c675`.
  - In the current single-cluster safety mode, `workloads/environments/prod/workloads/kustomization.yaml` is intentionally empty and `workloads/environments/prod/workloads-app.yaml` keeps `allowEmpty: true`, so those prod promote rows are not expected to reach `live` in this cluster.
  - Operator confirmation on 2026-03-10 states the promote path had already been exercised successfully before prod workloads were intentionally disabled, primarily because separate live prod workloads are not currently needed in this solo setup and the extra environment complicated observability.

Result: `L3.1`, `L3.2`, and `L3.3` are accepted as satisfied for the current single-cluster topology. The canonical 2026-03-10 evidence set now includes real `deploy` (`#46`), `promote` (`#47`), `config-change` (`#45`), and `rollback` (`#42`) rows, with an explicit operator-reviewed exception that current prod promote rows are traceability proof rather than live workload proof because prod workloads are intentionally disabled in single-cluster safety mode.

### 2026-03-11 Deploy-lock drill

Captured against `http://api.dev.homelab.local` on 2026-03-11:

- The portal backend was running the deployment-lock implementation backed by:
  - `apps/portal/backend/alembic/versions/20260310_0007_create_deployment_locks_table.py`
  - `apps/portal/backend/app/deployment_locks.py`
  - `apps/portal/backend/app/main.py`
  - `apps/portal/backend/app/deployment_reconciler.py`
- A manual `pending` deployment write for `homelab-api/dev` succeeded with:
  - `requestKey = "manual-lock-test-1"`
  - `action = "deploy"`
  - `status = "pending"`
- `GET /services/homelab-api?env=dev` returned a non-null `deploymentLock` for that in-flight mutation, including:
  - `serviceId = "homelab-api"`
  - `env = "dev"`
  - `requestKey = "manual-lock-test-1"`
  - `action = "deploy"`
  - `status = "pending"`
  - `lockedAt = "2026-03-11T11:29:40.121152+00:00"`
  - `expiresAt = "2026-03-11T11:59:40.121152+00:00"`
- While that lock was active, a second overlapping `pending` write for `homelab-api/dev` returned `HTTP 409 Conflict` with:
  - message: `Active deployment lock already exists for homelab-api/dev. Wait for the in-flight mutation to finish or clear its stale lock.`
  - the active lock payload embedded in the response body for operator diagnostics
- A terminal cleanup write for the same `requestKey` changed the first row to `status = "failed"` with `failureReason = "Intentional cleanup."`
- After that terminal write, `GET /services/homelab-api?env=dev` returned `deploymentLock = null`

Result: `L3.4` is satisfied. Deploy locks now exist per `serviceId + env`, overlapping writes are rejected with an operator-visible `409` payload, and locks release on terminal deployment state.

### 2026-03-11 Portal rollback workflow live evidence

Implementation landed in the portal codebase on 2026-03-11:

- `apps/portal/backend/app/github_workflows.py` dispatches the existing `gated-promotion.yml` workflow in `rollback` mode using GitHub `workflow_dispatch`.
- `apps/portal/backend/app/main.py` exposes `POST /rollbacks` for portal-initiated rollback requests.
- `apps/portal/.github/workflows/gated-promotion.yml` now accepts `operator_reason` and records it in rollback PR bodies plus workflow-written deployment records.
- `apps/portal/backend/app/deployment_reconciler.py` parses `- Reason:` lines from rollback PR bodies so operator-provided reason text survives reconcile/backfill.
- `apps/portal/frontend/src/pages/service-details-page.tsx` now exposes a rollback request panel for `homelab-api` and `homelab-web`.

Live drill captured on 2026-03-11:

- The first live `POST /rollbacks` request failed with `HTTP 503` because the running `homelab-api` Deployment did not yet have a GitHub Actions dispatch token.
- A temporary cluster-only secret, `homelab-api-github-actions`, was created in namespace `homelab-api`, and the live `homelab-api` Deployment was patched to consume `PORTAL_GITHUB_ACTIONS_TOKEN` from that secret so the portal rollback path could be exercised immediately.
- After that hotfix, a real portal rollback request for previous known-good tags (`sha-6beb5ac...`) returned `202 Accepted` and triggered GitHub Actions run `#22953368415` in `wlodzimierrr/homelab-portal`.
- Once the manual approval gate was approved, the workflow created `wlodzimierrr/homelab-workloads` PR `#53` (`automation/prod-rollback-image-update-22953368415`).
- Before closing the PR, the deployment-record API returned fresh `action = rollback` rows for both `homelab-api` and `homelab-web` in `prod` with:
  - `gitPrNumber = 53`
  - `gitRef = "automation/prod-rollback-image-update-22953368415"`
  - `deployReason = "Portal rollback fire drill for L3.6 traceability verification to previous known-good tags."`
  - a real `compareUrl`
- PR `#53` was then intentionally closed without merge as a safe fire drill. After `POST /deployments/reconcile?env=prod`, both rollback rows ended in:
  - `status = "failed"`
  - `failureReason = "GitOps pull request was closed without merge."`

Durability follow-up captured the same day:

- The temporary token wiring was moved into GitOps by adding a SOPS-managed dev secret, `homelab-api-github-actions`, to `workloads/apps/homelab-api/envs/dev`.
- Argo reconciled `homelab-api-dev` to workloads revision `9852145db62acd7c8c298612888441fbf2d30990`.
- `kubectl -n argocd get application homelab-api-dev -o json` then showed `Secret/homelab-api-github-actions` in the managed resource list with `status = "Synced"`.
- `kubectl -n homelab-api get deploy homelab-api -o jsonpath=...` confirmed the live API Deployment still reads `PORTAL_GITHUB_ACTIONS_TOKEN` from `homelab-api-github-actions` after the GitOps sync.
- A final no-op portal rollback request using the current prod tags returned `202 Accepted` at `2026-03-11T13:30:39.239012+00:00`, confirming that the durable GitOps-managed token path can still dispatch the rollback workflow after Argo adoption.

Result: `L3.6` is satisfied. Portal rollback is now first-class, traceable through deployment records and Git linkage, and backed by a GitOps-managed credential path rather than a cluster-only hotfix.

### 2026-03-11 Canonical service identity enforcement evidence

Captured against `http://api.dev.homelab.local` and the live `apps/portal` deployment on 2026-03-11:

- `GET /service-registry/diagnostics?env=dev` returned:
  - `identityDrift.driftCount = 0`
  - `identityDrift.okCount = 3`
  - canonical runtime rows for `homelab-api`, `homelab-web`, and support service `oauth2-proxy` with no violations
- `apps/portal/backend/scripts/live_catalog_validation.py` returned:
  - `status = "pass"`
  - `summary.identityDrift.driftCount = 0`
  - `summary.identityDrift.okCount = 3`
- Canonical write-path enforcement was confirmed by:
  - `POST /deployments` with `serviceId = "Homelab API"` returning `HTTP 422 Unprocessable Entity`
  - response message: `serviceId must use canonical lowercase-hyphen identity`
- CI-side enforcement was already wired before the live pass:
  - `workloads/scripts/check-service-identity-contract.sh`
  - `workloads/.github/workflows/validate-gitops-environment-contract.yml`

Result: `L3.5` is satisfied. Canonical service identity is now enforced through both CI and runtime diagnostics, and the live environment recorded a dated `identityDrift.driftCount = 0` pass.

### 2026-03-11 Deploy-window observability live evidence

Captured against `http://api.dev.homelab.local` on 2026-03-11 after the deploy-window observability backend/frontend changes were rolled out live:

- The live `homelab-api` Deployment was running image `ghcr.io/wlodzimierrr/homelab-api:sha-bd1d9a3d7066173796411bb029ee465254109ab1`.
- `GET /services/homelab-api/deployments?env=dev&limit=1` returned a fresh real deploy row:
  - `deploymentId = 043ff7f3-252c-48d5-baa7-5650e7e590ea`
  - `action = "deploy"`
  - `status = "live"`
  - `deployWindowStart = "2026-03-11T14:36:32+00:00"`
  - `deployWindowEnd = "2026-03-11T14:38:01.680655+00:00"`
  - `gitPrNumber = 54`
  - `compareUrl = "https://github.com/wlodzimierrr/homelab-portal/compare/e1eec1c85d94d8f8b274dbd82fbf8dc0e9a69a8e...bd1d9a3d7066173796411bb029ee465254109ab1"`
- `GET /services/homelab-api/observability/window?deploymentId=043ff7f3-252c-48d5-baa7-5650e7e590ea&logsPreset=errors&logsLimit=50` returned:
  - `context.windowSource = "deployment_record"`
  - `context.evidenceStatus = "resolved"`
  - `metricStatus = "ok"`
  - `timelineStatus = "ok"`
  - `logsStatus = "no_data"`
- The same deploy window queried through the explicit-window path,
  `GET /services/homelab-api/observability/window?windowStart=2026-03-11T14:36:32%2B00:00&windowEnd=2026-03-11T14:38:01.680655%2B00:00`,
  returned:
  - `context.windowSource = "explicit_window"`
  - `context.evidenceStatus = "resolved"`
  - `metricStatus = "ok"`
  - `timelineStatus = "ok"`
  - `logsStatus = "no_data"`

Interpretation:

- Deploy-window queries are now anchored to persisted deployment windows rather than inferred release/runtime timestamps.
- `logsStatus = "no_data"` is surfaced as a resolved no-telemetry result, not as provider failure and not as missing deployment-window evidence.
- The same time window is reusable through both a first-class deployment record and a raw explicit time range.

Result: `L3.7` is satisfied. Deploy-window observability is now anchored to deployment records and live-verified through both deployment-ID and explicit-window queries.

### 2026-03-11 Deployment timeline filter completion

Captured against the current `apps/portal` frontend code on 2026-03-11:

- `apps/portal/frontend/src/lib/adapters/deployments.ts` continues to use first-class deployment records as the source of truth for timeline rows.
- `apps/portal/frontend/src/pages/service-deployments-page.tsx` already rendered mixed-action deployment history from live rows:
  - deploy `#46`
  - promote `#47`
  - config-change `#45`
  - rollback `#42`
- The page was extended with explicit:
  - action filter
  - status filter
  - existing service/env scope
  - existing sort modes (`newest`, `worst_impact`)
- The production frontend build succeeded after the filter change:
  - `npm run build`
  - output bundle included the updated deployments page without TypeScript or Vite errors

Interpretation:

- Deployment timeline rows now come from deployment records rather than inferred runtime history.
- Mixed-action history is visible in one place with operator-usable action/status filtering.
- Service and environment remain explicit scope dimensions through the service deployments route itself.

Result: `L3.8` is satisfied. The deployment history timeline now exposes mixed-action records with reason/changelog context and explicit filtering/sorting controls.

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
- **Status:** DONE (2026-03-09)
- **Problem:** Service details still show placeholder status/version states (`Health: unknown`, `Sync: unknown`, deployed version `N/A`) and deployment history can be empty or unavailable despite live catalogs being healthy.
- **Evidence:** This was closed by restoring live metadata on both the backend and frontend. On 2026-03-09, live `GET /services/homelab-api?env=dev` returned `version = "sha-d642ac..."`, `health = "healthy"`, and `sync = "synced"`, while `GET /services/homelab-api/deployments?env=dev` returned a real deployment row. The service-details UI also now falls back to `/releases?serviceId=...` so overview cards no longer depend on placeholder-only service payloads.
- **Acceptance Criteria:**
  - Service details status badges are backed by live release/Argo/service data.
  - Deployed version resolves from live release/deployment metadata instead of placeholders.
  - Recent deployment history loads consistently for live services.
- **Risk:** Medium

### P1.7 Restore live logs quickview behavior on service details
- **Status:** DONE (2026-03-09)
- **Problem:** Logs panels still degrade to “logs unavailable” or empty-state behavior even when Loki is healthy.
- **Evidence:** After the LogQL preset fix rolled out on 2026-03-09, live `GET /services/homelab-web/logs/quickview?preset=warnings&range=1h` and `...preset=restarts...` returned `200` with `providerStatus.status = "healthy"` and empty `lines`, not `502` / parse errors. That means the UI can now show a legitimate no-results state instead of a broken logs state.
- **Acceptance Criteria:**
  - Service details logs quickview loads bounded lines for a known live service when Loki is healthy.
  - User-visible errors distinguish `no matching log lines` from request/config/provider failures.
  - Console/network errors for logs quickview are eliminated for healthy providers.
- **Risk:** Medium

### P1.8 Reconcile live service count and endpoint visibility in the portal
- **Status:** DONE (2026-03-09)
- **Problem:** Live cluster diagnostics report three services, but some portal surfaces still appear to show only two or provide `No public/internal endpoints available` for live services.
- **Evidence:** Live `/services?env=dev` continues to expose `homelab-api`, `homelab-web`, and `oauth2-proxy`, and the portal work narrowed the remaining UI problem to deployment-history provenance rather than dropped service rows or endpoint-visibility confusion. Service-only rows such as `oauth2-proxy` are now intentionally preserved and rendered as live cluster services.
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
- **Status:** DONE (2026-03-09)
- **Problem:** The services list still renders `Health: unknown`, `Sync: unknown`, and `Last Deploy: N/A` for live services even though cluster sync and service details now resolve real metadata.
- **Evidence:** On 2026-03-09 the Services page still showed `Health: unknown`, `Sync: unknown`, and `Last Deploy: N/A` for `homelab-api` and `homelab-web`, while `GET /services/homelab-api?env=dev` returned `health = "healthy"`, `sync = "synced"`, and `version = "sha-d642ac..."`.
- **Evidence (Resolved):** The services-list adapter now enriches rows from live `/releases` traceability data plus metrics summary data, and the remaining unresolved portal issue on 2026-03-09 narrowed to deployment-history provenance rather than list-level `health/sync/last deploy` placeholders. This ticket is therefore closed as list enrichment work, with older deployment-history depth tracked separately below and in `homelab.md`.
- **Acceptance Criteria:**
  - Services list rows reflect live health/sync metadata instead of default unknown badges when the service detail endpoint has real values.
  - Last deploy is populated from live deployment/release metadata where available.
  - Services list cards no longer show false `No Data` uptime states when live metrics exist for the service.
- **Risk:** Medium

### P1.11 Restore service details overview cards from live metadata
- **Status:** DONE (2026-03-09)
- **Problem:** The Service Overview tab still shows `Deployed Version: N/A` and `Service Status: unknown` for some live services even after backend metadata restoration.
- **Evidence:** On 2026-03-09 the `homelab-web` service page still rendered `Deployed Version: N/A` and `Health: unknown / Sync: unknown` in the overview cards, despite the service-details backend work and live service metadata improvements already shipping for `homelab-api`.
- **Evidence (Resolved):** The service-details frontend now uses `/releases?serviceId=...` as a live fallback source for version, health/sync, and deployment cards. By the end of the 2026-03-09 cleanup, the only remaining service-details gap was deployment-history provenance, not the overview cards themselves.
- **Acceptance Criteria:**
  - Overview cards on the Service page render live deployed version, health, and sync values for both `homelab-api` and `homelab-web`.
  - Placeholder copy is replaced by accurate live-state messaging or a precise missing-data reason.
  - Frontend identity/adapters use the resolved service detail payload instead of stale fallback values.
- **Risk:** Medium

### P1.12 Restore service-details observability embeds and quick links
- **Status:** DONE (2026-03-09)
- **Problem:** The Service page still shows `Grafana unavailable` and unavailable Argo/Grafana quick links, leaving latency/error panels and external navigation degraded even when backend observability data is available.
- **Evidence:** On 2026-03-09 the service page still rendered `Grafana embed URL is not configured`, `Argo CD Application unavailable due to missing monitoring URL configuration`, and `Grafana Dashboard unavailable due to missing monitoring URL configuration`.
- **Evidence (Resolved):** The frontend now derives Argo links from resolved `argoAppName` values and infers the real homelab Argo/Grafana base URLs on homelab hosts, so service-details quick links and embed configuration are no longer stuck in a permanent missing-config state. Follow-on polish for scoped Grafana/Loki drill-downs and the portal-native logs/trend presentation was completed in `P1.14`.
- **Acceptance Criteria:**
  - Latency & Error Trends panels render usable embeds or a precise intentional disabled-state tied to missing config.
  - Quick links for Argo CD, Grafana dashboard, and logs resolve correctly when URLs are configured.
  - UI copy distinguishes missing operator configuration from provider/data failures.
- **Risk:** Low

### P1.13 Restore recent deployments panel and deployment history on service details
- **Status:** DONE (2026-03-09)
- **Problem:** The Service page still reports `Deployment history unavailable` or empty recent deployments even when live `/services/{serviceId}/deployments` data exists.
- **Evidence:** On 2026-03-09 the Service page still showed `Deployment history unavailable` / `Service endpoint is not available in this backend` for Recent Deployments, despite `GET /services/homelab-api/deployments?env=dev` returning a live deployment row with version and deployed timestamp.
- **Evidence (Resolved):** The service page and the full deployments page now render the current live deployment row instead of reporting backend unavailability. When comparison snapshots are missing, the UI now explains that Prometheus has no retained samples for that deployment window rather than presenting a broken state. True multi-release historical deployment reconstruction is deferred to future roadmap items in `homelab.md` (`T6.7.4` to `T6.7.6`).
- **Acceptance Criteria:**
  - Recent Deployments on the Service page renders live deployment rows when `/services/{serviceId}/deployments` returns data.
  - “Open full history” routes to a working deployment history experience for the service.
  - Error states distinguish missing backend support from transient request failures.
- **Risk:** Medium

### P1.14 Make Grafana trend panels and full logs deep-links land on useful scoped views
- **Status:** DONE (2026-03-11)
- **Problem:** The service-details page needed useful scoped drill-downs for trends and full logs, but brittle Grafana iframe behavior made the page depend on a visualization path that was not reliable across services.
- **Evidence:** On 2026-03-11, the service-details page was completed with portal-native `P95 Latency Trend` and `Error Rate Trend` cards driven by the backend metrics fallback logic instead of iframe-only Grafana embeds, while keeping scoped `Open in Grafana` drill-downs built from resolved `serviceIdentity` metadata (`serviceId`, `namespace`, `appLabel`, `env`, `argoAppName`, and time range). The page also gained a proper console-style logs viewer above the rollback panel, defaulting to `All logs` with optional `Errors`, `Warnings`, and `Restarts` filters, while preserving the older logs panel below. `apps/portal/frontend/src/pages/service-details-page.tsx`, `apps/portal/frontend/src/lib/adapters/logs-quickview.ts`, `apps/portal/backend/app/logs_quickview.py`, and `apps/portal/backend/app/main.py` were updated, and `npm run build`, `python -m compileall app scripts`, and `python scripts/generate_openapi.py` all passed.
- **Acceptance Criteria:**
  - Native latency and error trend cards render from portal/backend data for the selected service and time range, without relying on fragile iframe embeds.
  - “Open in Grafana” and “Open full logs” resolve to useful scoped Grafana/Loki views for the current service and selected window.
  - Grafana panel and logs URLs use resolved service metadata (`serviceId`, `namespace`, `appLabel`, and where relevant `argoAppName`) instead of stale frontend fallbacks.
  - If a deep-link cannot be built, the UI explains which required URL template or variable is missing.
- **Risk:** Low

### P1.15 Evaluate public routable portal/API hosts for external automation
- **Status:** TODO
- **Problem:** GitHub-hosted automation cannot reliably call private `.homelab.local` endpoints, which makes direct workflow-to-portal API writes brittle and blocks future webhook-style integrations.
- **Evidence:** On 2026-03-10, `wlodzimierrr/homelab-portal` Actions run `#22904757614` failed in the `Record deployment requests in portal backend` step with `curl: (6) Could not resolve host: api.dev.homelab.local`. Current ingress hosts are private-only (`api.dev.homelab.local`, `portal.dev.homelab.local`), so GitHub-hosted runners cannot resolve them.
- **Acceptance Criteria:**
  - A decision is recorded for one of these paths:
    - keep private-only ingress and rely on backend reconciliation for correctness, or
    - expose a public portal host, or
    - expose a narrowly scoped public API host for automation/webhooks.
  - If a public host is adopted, DNS, TLS, auth, and ingress policy are configured and documented.
  - GitHub-hosted automation can reach the chosen endpoint successfully from Actions.
  - The chosen design documents security boundaries clearly, especially for any public API path.
- **Likely Work Areas:**
  - Public DNS/ingress hostnames under `*.wlodzimierrr.co.uk`.
  - OAuth/UI auth for the portal vs. machine-to-machine auth for API endpoints.
  - Rate limiting, narrow endpoint exposure, and optional IP allowlisting for automation paths.
  - Updating workflow secrets and runbooks if a public endpoint becomes the preferred automation target.
- **Risk:** Medium

### P1.16 Plan single-cluster dual-environment isolation
- **Status:** TODO
- **Problem:** The current repo intentionally keeps prod workloads disabled in single-cluster safety mode, but there is no concrete design ticket for how dev and prod should coexist safely on one cluster if prod workloads are re-enabled later.
- **Evidence:** `workloads/README.md`, `docs/architecture/deployment-flow.md`, and `docs/homelab.md` all state that `environments/prod/workloads` is intentionally empty and `allowEmpty: true` is kept to avoid accidental recreation of live prod apps in the same cluster. On 2026-03-10, the operator confirmed that separate live prod workloads were turned off because they were not currently needed and created extra observability complexity in a solo setup.
- **Acceptance Criteria:**
  - A written decision exists for whether to keep `prod` as Git intent only or to re-enable live prod workloads on the same cluster.
  - If dual-env single-cluster mode is chosen, the design documents namespace layout, ingress hostnames, data separation, and resource isolation rules.
  - The design names explicit prerequisites before prod workloads may be re-enabled.
  - The decision is reflected in runbooks and GitOps structure so future changes do not drift from the chosen model.
- **Likely Work Areas:**
  - Namespace strategy such as `homelab-api-dev` / `homelab-api-prod` and `homelab-web-dev` / `homelab-web-prod`.
  - Shared-platform vs. per-env workload boundaries.
  - Separate secrets, databases, backups, and ingress hosts for each env.
  - ResourceQuota, LimitRange, affinity, and NetworkPolicy guardrails between dev and prod.
- **Risk:** Medium

### P1.17 Add env-scoped observability and isolation guardrails for dual-env mode
- **Status:** TODO
- **Problem:** The main reason prod workloads were disabled was observability friction. If dev and prod ever run together again, dashboards, alerts, and logs must separate envs cleanly or the extra environment will keep creating operational noise.
- **Evidence:** Operator confirmation on 2026-03-10 states that live prod workloads were switched off largely because the extra environment complicated observability. Existing docs already note that the current single-cluster mode keeps prod disabled to avoid accidental overlap and operational confusion.
- **Acceptance Criteria:**
  - Metrics, logs, and alerts are filterable and trustworthy by `env=dev|prod` for every canonical service.
  - Grafana dashboards, Loki queries, and Alertmanager routes distinguish dev from prod without manual guesswork.
  - Service identity and namespace conventions are reflected consistently in Prometheus labels, Loki labels, and alert metadata.
  - A short drill proves that a prod-only signal does not appear as ambiguous dev noise, and vice versa.
- **Likely Work Areas:**
  - Prometheus/Loki label normalization for `service`, `env`, `namespace`, and `argoAppName`.
  - Dashboard variables and alert templates that make env separation explicit.
  - Namespace and ingress naming conventions that match the observability labels.
  - Runbook updates for debugging when both envs coexist on one cluster.
- **Risk:** Medium

### P1.18 Re-enable prod workloads in single-cluster mode behind explicit guardrails
- **Status:** TODO
- **Problem:** The repo already contains prod overlay intent and gated promotion workflows, but there is no controlled ticket for re-enabling real prod workloads once the operator decides the cluster should host both envs again.
- **Evidence:** `workloads/environments/prod/workloads-app.yaml` keeps `allowEmpty: true`, `workloads/environments/prod/workloads/kustomization.yaml` is intentionally empty, and `workloads/README.md` explicitly says `homelab-api-prod` / `homelab-web-prod` should remain absent unless a safe prod target exists.
- **Acceptance Criteria:**
  - Prod workloads can be enabled intentionally by reverting the current single-cluster safety guardrail in a documented way.
  - Prod apps use isolated namespaces, secrets, ingress hosts, and data paths.
  - Promotion and rollback evidence can reach real `promote -> live` and `rollback -> live` outcomes without colliding with dev.
  - Observability, quotas, and network policies are validated before the guardrail is removed.
- **Likely Work Areas:**
  - Populate `environments/prod/workloads/kustomization.yaml` with the intended prod apps.
  - Remove or narrow `allowEmpty: true` only when the target is actually ready.
  - Create prod namespace overlays and env-specific data/secret wiring.
  - Run a staged enablement: one service first, then full prod workloads.
- **Risk:** High

### P1.19 Define a universal service observability contract
- **Status:** DONE (2026-03-11)
- **Problem:** The platform needed an explicit service observability contract so future services could declare one authoritative HTTP metrics mode instead of relying on implicit backend fallback behavior.
- **Evidence:** On 2026-03-11, live inspection showed `homelab-api` returning populated `p95LatencyMs` and `errorRatePct` because it has app-level Prometheus instrumentation plus a `ServiceMonitor`, while `homelab-web` returned only `uptimePct` and `restartCount`. The current backend metrics logic in `apps/portal/backend/app/main.py` already tries app metrics first and Traefik fallback second, but `workloads/apps/homelab-web/base/deployment.yaml` has no metrics endpoint or `ServiceMonitor`, and there is no repo-level declaration saying whether a service is `app-native`, `ingress-derived`, or `no-http`.
- **Acceptance Criteria:**
  - A written observability contract defines the required baseline for every service: canonical labels/identity, logs, uptime, restart visibility, deploy-window visibility, and one declared HTTP metrics mode.
  - The contract supports at least three modes:
    - `app-native` for services that expose their own HTTP metrics,
    - `ingress-derived` for routed HTTP services that rely on edge metrics,
    - `no-http` for background/support services where latency/error rate is not a primary signal.
  - The chosen mode is declared in Git for each service and becomes part of service scaffolding/registration.
  - Portal APIs and diagnostics can report which observability mode a service is using.
- **Likely Work Areas:**
  - Add an ADR or contract doc for service observability modes.
  - Extend service catalog or scaffold metadata to record the selected mode.
  - Align portal diagnostics and UI wording with the declared mode.
- **Progress Notes (2026-03-11):**
  - Added [service-observability.md](/home/wlodzimierrr/homelab/docs/contracts/service-observability.md) defining the universal baseline plus `app-native`, `ingress-derived`, and `no-http` modes.
  - Added `observability.mode` declarations to [services.yaml](/home/wlodzimierrr/homelab/workloads/services.yaml) for `homelab-api` and `homelab-web`.
  - Extended scaffold and CI guardrails so new services must declare an observability mode.
  - Added portal-side persistence and diagnostics plumbing so Git-backed project metadata can report the declared observability mode after deploy.
  - Verified the contract path with:
    - `ruff check app tests`
    - `pytest tests/test_gitops_project_sync.py tests/test_service_identity_validation.py -q`
    - `./scripts/check-service-identity-contract.sh`
    - `./scripts/smoke-test-scaffold-generator.sh`
    - `python scripts/generate_openapi.py`
    - `python -m compileall app tests scripts`
    - `npm run build`
- **Risk:** Medium

### P1.20 Establish a universal ingress-derived HTTP metrics baseline
- **Status:** DONE (2026-03-11)
- **Problem:** The platform needed a universal HTTP metrics baseline for routed services that do not expose app-native Prometheus metrics.
- **Evidence:** On 2026-03-11, direct inspection of the live Traefik metrics endpoint on `:9100` confirmed `traefik_service_requests_total` and `traefik_service_request_duration_seconds_bucket` were being exported with service labels like `homelab-web-homelab-web-80@kubernetes` and `homelab-api-homelab-api-80@kubernetes`. Prometheus is configured to scrape `PodMonitor`s labeled `release=kube-prometheus-stack`, and after validating the Traefik scrape path and generating live traffic to `portal.dev.homelab.local`, the portal returned ingress-derived fallback data for `homelab-web`: `GET /services/homelab-web/metrics/summary?range=24h` returned `uptimePct = 100`, `p95LatencyMs = 95`, `errorRatePct = 0`, `restartCount = 0`, and `GET /services/homelab-web/metrics/trends?range=24h` reported `querySource = "traefik_fallback"` for both latency and error-rate series.
- **Acceptance Criteria:**
  - The ingress layer exports request-count and request-duration metrics into Prometheus for every routed HTTP service.
  - The exported labels can be mapped deterministically to canonical portal service identity (`serviceId`, `namespace`, `appLabel`, optionally `argoAppName`).
  - A routed service with no app-native metrics still shows live latency and error-rate data through the portal.
  - At least one newly scaffolded HTTP service can rely only on the ingress-derived mode and still surface useful portal metrics.
- **Likely Work Areas:**
  - Preserve the Traefik scrape path and label mapping as a platform baseline for future routed services.
  - Keep portal fallback PromQL aligned with the real Traefik `service` label shape.
  - Use `homelab-web` as the reference proof case for `ingress-derived` services until a newly scaffolded routed service is added.
- **Risk:** High

### P1.21 Enforce per-service metrics mode in CI and runtime diagnostics
- **Status:** IN PROGRESS (2026-03-11)
- **Problem:** Once multiple observability modes exist, the platform needs hard validation so new services cannot silently land in an ambiguous state where the portal says `No data` but the root cause is undeclared or misconfigured telemetry.
- **Evidence:** On 2026-03-11, the observability-mode enforcement path was implemented locally across both repos. In `workloads`, `scripts/check-service-identity-contract.sh` now fails `app-native` services that lack a `Service` or `ServiceMonitor` and fails `ingress-derived` services that lack canonical Traefik ingress manifests, while `scripts/scaffold-service.py` and `scripts/smoke-test-scaffold-generator.sh` were updated so newly scaffolded `python-fastapi` services emit a base `servicemonitor.yaml` and still pass the stricter contract checks. In `apps/portal`, `backend/app/service_observability.py`, `backend/app/main.py`, and `backend/app/observability_config.py` now emit mode-aware diagnostics for metrics summary/trends and health timeline fallback, and the service details UI now prefers those actionable observability messages over a generic `No data` banner. Verified locally with:
  - `./.venv/bin/ruff check app tests`
  - `./.venv/bin/pytest tests/test_service_observability.py tests/test_gitops_project_sync.py tests/test_service_identity_validation.py -q`
  - `./.venv/bin/python scripts/generate_openapi.py`
  - `./.venv/bin/python -m compileall app tests scripts`
  - `./scripts/check-service-identity-contract.sh`
  - `./scripts/smoke-test-scaffold-generator.sh`
  - `npm run build`
- **Acceptance Criteria:**
  - CI fails when a service declares `app-native` but lacks the required scrape path/ServiceMonitor, or declares `ingress-derived` but lacks ingress/route metadata.
  - Runtime diagnostics distinguish:
    - expected unsupported metrics,
    - no retained data,
    - misconfigured or missing telemetry source.
  - Scaffolded services must choose an observability mode explicitly.
  - Portal operator diagnostics surface actionable mismatch reasons instead of generic `No data`.
- **Likely Work Areas:**
  - Extend GitOps/service catalog validation scripts.
  - Add backend diagnostics for observability mode and telemetry-source health.
  - Update service-details UI to explain why a metric card is empty.
- **Risk:** Medium

### P1.22 Backfill existing services onto the observability contract and capture proof
- **Status:** TODO
- **Problem:** Even with a universal contract, the current services still need to be classified and proven against it, otherwise future services will inherit a model that has never been exercised across mixed service types.
- **Evidence:** `homelab-api` already behaves like an `app-native` service, `homelab-web` behaves like an `ingress-derived` candidate, and `oauth2-proxy` behaves like a support/no-http service. That makes the current repo a good mixed proof set, but no ticket currently captures the migration and evidence work explicitly.
- **Acceptance Criteria:**
  - `homelab-api`, `homelab-web`, and `oauth2-proxy` are each mapped to an explicit observability mode in Git.
  - Live portal diagnostics and service pages show expected behavior for all three modes.
  - At least one dated evidence note records successful live checks for each mode.
  - The resulting pattern is documented as the standard path for future services added in Phase 6 and later.
- **Likely Work Areas:**
  - Update existing service metadata.
  - Add one dated live validation pass per service/mode.
  - Refresh runbooks and scaffolding docs to use the backfilled services as reference examples.
- **Risk:** Medium

## Suggested execution order

1. P0.1 GitOps repo access
2. P0.2 Cluster and Argo sync connectivity
3. P0.3, P0.4, P0.5 monitoring provider reachability
4. P1.1 CronJob green run verification
5. P1.2 portal UI rebaseline
6. P1.3 end-to-end validation rerun
7. P1.4 metrics coverage
8. P1.19 universal service observability contract
9. P1.21 per-service metrics mode enforcement and diagnostics
10. P1.22 backfill current services onto the observability contract
11. P1.5 timeline window contract
12. P1.15 public portal/API host evaluation for external automation
13. P1.16 single-cluster dual-env isolation plan
14. P1.17 env-scoped observability and isolation guardrails
15. P1.18 guarded prod re-enablement in single-cluster mode
16. P2.2 intermittent Postgres reachability during scheduled syncs
