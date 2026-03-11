# Portal Level 3 Readiness Audit

Use this runbook before starting any Portal Level 3 (`T6.x`) work. The outcome is binary:

- `GO`: every readiness gate passes and the Render-like checklist has no `FAIL` rows.
- `NO-GO`: any gate or checklist item still fails.

As of 2026-03-10, the expected result is `NO-GO`.

## Audit cadence

Run this audit:

- before opening the first Level 3 implementation PR
- after any major delivery-path change
- after each rollback or promotion fire drill
- weekly while preparing for Level 3

Record the latest result in `docs/homelab.md` and attach dated evidence in `docs/homelab-issues.md`.

## Evidence rules

Only count evidence that is dated and reproducible:

- API responses captured with timestamps
- workflow run URLs or screenshots
- PR URLs and merge SHAs
- Argo sync timestamps
- runbook notes naming the exact service and environment

Do not mark an item complete based on "the workflow exists" alone when the gate requires live proof over time.

## Readiness gates

### 1. Multi-service stability

Pass only if all are true:

- at least 3 canonical services are live in the target environment
- they remain healthy and synced for 14-28 days
- no force-sync or manual cluster intervention was required during that window

Suggested evidence:

- `GET /services?env=dev`
- `GET /service-registry/diagnostics?env=dev`
- `GET /releases?env=dev&limit=50`
- dated operator notes showing no manual Argo or `kubectl` recovery was needed

Current status:

- Cleared on 2026-03-09 by operator confirmation: `homelab-api`, `homelab-web`, and `oauth2-proxy` have been deployed and stable for at least 14 days, implying a stable window starting on or before 2026-02-23, with no force-sync or manual cluster intervention required.

### 2. Immutable artifacts from CI

Pass only if all are true:

- every merged commit publishes an immutable image tag such as `sha-<commit>`
- image metadata is traceable back to the source commit
- the build output is reproducible and auditable

Repo prerequisites already present:

- `apps/portal/.github/workflows/portal-images.yml`
- `docs/architecture/deployment-flow.md`

Suggested evidence:

- three recent `main` merges with matching GHCR tags
- matching GitOps PRs opened by the image-publish workflow
- retained SBOM/provenance artifacts for those runs

Current status:

- Cleared on 2026-03-09: three recent successful `portal-images.yml` runs on `main` were captured with matching GitOps PRs (`#33`, `#34`, `#35`), retained SBOM artifacts, and authenticated GHCR manifest checks returning `ok` for `homelab-api` and `homelab-web` on all three commit SHAs.

### 3. Proven image promotion

Pass only if one of these is true:

- dev auto-promotion succeeded for more than 10 consecutive merges
- manual PR-based bump/promotion was tested end-to-end at least 3 times with zero failed syncs

Repo prerequisites already present:

- `apps/portal/.github/workflows/portal-images.yml`
- `apps/portal/.github/workflows/gated-promotion.yml`
- `docs/runbooks/gitops-dev-prod-promotion.md`

Suggested evidence:

- list of consecutive workflow runs with result, PR URL, merge SHA, and Argo outcome
- for rollback drills, record `N -> N+1 -> N` tag movement and exact reconcile timing

Current status:

- Cleared on 2026-03-09 by operator-reviewed evidence: recent dev auto-bump runs were matched to `homelab-workloads` PRs and Argo deployment timestamps for `homelab-api-dev` and `homelab-web-dev`. PR `#34` was intentionally left unmerged after the operator forgot to merge it before a newer push; it was accepted as a skipped/superseded promotion rather than a GitOps or Argo failure.

### 4. Secrets strategy chosen and in-use

Pass only if all are true:

- SOPS or Sealed Secrets is the chosen standard
- secrets committed to Git are encrypted at rest
- at least 2 services consume decrypted secrets successfully through the delivery path

Repo prerequisites already present:

- `docs/runbooks/sops-secrets.md`
- `docs/runbooks/secret-rotation-quarterly.md`
- `ansible/roles/argocd/tasks/main.yml` repo-server `ksops` patch
- encrypted secret generators in the `homelab-api` and `homelab-web` overlays

Suggested evidence:

- encrypted secret manifests in Git
- one dated Argo/Kubernetes validation where `homelab-api` and `oauth2-proxy` reconcile from Git-managed encrypted secrets with no manual secret apply
- pod startup evidence for at least two services

Current status:

- Cleared on 2026-03-09: the Argo repo-server `ksops`/age-key patch was applied live, `homelab-workloads` revision `5d3095f` reconciled successfully, and both `homelab-api-dev` and `homelab-web-dev` rendered real `Secret` objects while remaining `Synced` and `Healthy`.

### 5. Observability baseline active

Pass only if all are true:

- Prometheus scrapes metrics
- Grafana dashboards are usable
- Loki logs are queryable
- at least one real alert fired and was confirmed

Repo prerequisites already present:

- `docs/runbooks/monitoring-kube-prometheus-stack.md`
- `docs/runbooks/logging-loki-promtail.md`
- `docs/runbooks/monitoring-provider-readiness.md`

Suggested evidence:

- `GET /monitoring/providers/diagnostics`
- `GET /alerts/active`
- `GET /monitoring/incidents`
- one dated alert drill or incident with dashboard/log links

Current status:

- Cleared on 2026-03-09: a temporary `PrometheusRule` drill (`HomelabObservabilityDrill`) fired for `homelab-api` in `dev`, was confirmed in Prometheus and Alertmanager, appeared in the portal alert feed via `GET /alerts/active?env=dev&serviceId=homelab-api&limit=50`, shared the same evidence window as healthy `GET /services/homelab-api/metrics/summary?range=1h`, and cleared cleanly after the rule was deleted.

### 6. Runbook for rollback exists and tested

Pass only if all are true:

- the rollback path is documented
- a real rollback was rehearsed against the delivery path
- Argo applied the rollback within 5 minutes

Repo prerequisites already present:

- `docs/runbooks/gitops-dev-prod-promotion.md`

Suggested evidence:

- rollback PR URL or workflow run URL
- Argo application sync timestamps before and after rollback
- service verification showing the known-good version is live again

Current status:

- Cleared on 2026-03-09: `homelab-workloads` revert commit `b11ba312617de122359916a13a16fde3a3dbee86` rolled `homelab-api-dev` and `homelab-web-dev` from `sha-cd76e59...` back to the previous known-good `sha-6beb5ac...`. The revert was pushed at `18:22:38Z`, and both apps were `Synced`, `Healthy`, and running the restored images by `18:24:54Z`, for a measured Argo recovery time of `2m16s`.

## Render-like checklist

These checks guard Phase 6 scope. As of 2026-03-11, the readiness checklist is fully satisfied for the current single-cluster topology.

### Deployment records exist for every mutation

Pass when:

- an Alembic migration adds a first-class `deployments` table
- deploy/promote/rollback/config-change flows all write rows there
- read APIs return those rows as the source of truth

Current status:

- Cleared on 2026-03-10: `deployments` table persistence, PR-keyed reconcile/write paths, and deployment-record-backed APIs are live. Fresh dated evidence exists for a real dev deploy row (`#46` -> `live`), a real dev config-change row (`#45` -> `live`), a real prod rollback row (`#42` -> `failed` after the PR was intentionally closed without merge), and a real prod promote row (`#47`) for both services.
- Single-cluster safety-mode note: `workloads/environments/prod/workloads/kustomization.yaml` is intentionally empty and `workloads/environments/prod/workloads-app.yaml` keeps `allowEmpty: true`, so current promote proof is accepted as row creation and traceability rather than live prod workload convergence in this cluster.

### Deploy lifecycle is visible

Pass when:

- deployment rows transition through `pending`, `deploying`, `live`, or `failed`
- the portal shows those states directly

Current status:

- Cleared on 2026-03-10: live deployment records now show `pending`, `deploying`, `live`, and `failed` states with timestamps and failure text. Dated evidence includes rollback `#42` as `pending` before close and `failed` after close, deploy `#46` as `deploying` before later becoming `live`, and config-change `#45` as `live`.

### Post-merge rollout verification is automatic

Pass when:

- a watcher links merged PRs to exactly one deployment record
- Argo and Kubernetes rollout results update that record

Current status:

- Cleared on 2026-03-10: the live reconciler updates real rows automatically after GitOps state changes. Fresh evidence shows deploy `#46` verified to `live`, config-change `#45` verified to `live`, rollback `#42` verified to `failed` with operator-visible reason text after the PR was closed without merge, and promote `#47` producing merged prod `promote` rows for both services.
- Single-cluster safety-mode note: because the prod workloads path is intentionally empty here, current promote verification is accepted as operator-reviewed merged-PR detection plus deployment-row progression, not as a live prod workload rollout in this cluster.

### Deploy-scoped logs and metrics are anchored to deploy windows

Pass when:

- the portal timeline uses deployment-record windows, not inferred release timestamps
- before/after metrics and logs are tied to the same deployment ID

Current status:

- Cleared on 2026-03-11:
  - `apps/portal/backend/app/main.py` serves `/services/{service_id}/observability/window` and resolves deployment windows from persisted deployment records or explicit `windowStart/windowEnd`.
  - Deployment snapshots are anchored to `deployWindowStart` / `deployWindowEnd`, not a synthetic completion timestamp.
  - `apps/portal/frontend/src/pages/service-deployments-page.tsx` renders a deployment-scoped observability drilldown for the selected deployment record.
  - A live deploy-window drill against `homelab-api/dev` deployment `043ff7f3-252c-48d5-baa7-5650e7e590ea` (deploy `#54`) returned:
    - `context.evidenceStatus = "resolved"`
    - `metricStatus = "ok"`
    - `timelineStatus = "ok"`
    - `logsStatus = "no_data"`
  - The same deploy window, queried through explicit `windowStart/windowEnd`, returned the same resolved metrics/timeline result and the same explicit `logsStatus = "no_data"`, which confirms missing telemetry is surfaced separately from query failure and from missing deployment-window evidence.

### Deployment history timeline shows mixed actions with explicit filters

Pass when:

- deployment history rows come from deployment records, not inferred runtime data
- the timeline shows deploy/promote/config-change/rollback rows together
- the UI can filter and sort by the required timeline dimensions

Current status:

- Cleared on 2026-03-11:
  - `apps/portal/frontend/src/lib/adapters/deployments.ts` preserves deployment-record action, actor, reason, failure text, compare links, and PR metadata.
  - `apps/portal/frontend/src/pages/service-deployments-page.tsx` renders requested/completed timestamps, action, version, lifecycle state, Argo sync/health, PR links, compare links, and deploy reason directly from deployment records.
  - The live deployment history already contains mixed-action evidence:
    - deploy `#46`
    - promote `#47`
    - config-change `#45`
    - rollback `#42`
  - The page now exposes explicit action and status filters on top of the existing service/env scope and sorting modes, and the production frontend build completed successfully after that change.

### Canonical service identity is enforced end-to-end

Pass when:

- portal registry, Argo app names, Kubernetes labels, Prometheus selectors, Loki selectors, and release joins all validate against one canonical contract
- CI fails when that mapping drifts

Current status:

- Cleared on 2026-03-11:
  - `apps/portal/backend/app/service_identity.py` and `apps/portal/backend/app/service_identity_validation.py` enforce the canonical contract at runtime
  - `GET /service-registry/diagnostics?env=dev` returned `identityDrift.driftCount = 0` and `okCount = 3`
  - `apps/portal/backend/scripts/live_catalog_validation.py` returned `status = "pass"` with `summary.identityDrift.driftCount = 0`
  - `POST /deployments` with non-canonical `serviceId = "Homelab API"` returned `HTTP 422`
  - `workloads/scripts/check-service-identity-contract.sh` remains wired into `validate-gitops-environment-contract.yml`

### Deploy locks exist per service and environment

Pass when:

- overlapping mutations on the same `serviceId + env` are rejected consistently
- stale locks are cleaned up

Current evidence:

- `apps/portal/backend/alembic/versions/20260310_0007_create_deployment_locks_table.py` persists one lock row per `serviceId + env`
- `apps/portal/backend/app/main.py` rejects conflicting `POST /deployments` writes with `409 Conflict` and returns the active lock payload
- `apps/portal/backend/app/deployment_reconciler.py` and `apps/portal/backend/app/deployment_locks.py` refresh/release matching locks and run stale cleanup
- live drill on 2026-03-11 confirmed:
  - manual `pending` write created a lock for `homelab-api/dev`
  - `GET /services/homelab-api?env=dev` returned a non-null `deploymentLock`
  - overlapping `pending` write returned `409 Conflict`
  - terminal `failed` write released the lock and `deploymentLock` returned to `null`

### Rollback is available from the portal and remains traceable

Pass when:

- the portal can initiate rollback through the Git-backed path
- the resulting deployment record stores operator reason, compare link, and rollout result

Current status:

- Cleared on 2026-03-11: a portal-initiated rollback fire drill used `POST /rollbacks`, triggered `wlodzimierrr/homelab-portal` workflow run `#22953368415`, created `wlodzimierrr/homelab-workloads` PR `#53`, and produced real `action = rollback` prod rows for both `homelab-api` and `homelab-web` with operator reason, PR link, `compareUrl`, and final reconciled `failed` status after the PR was intentionally closed without merge.
- The temporary cluster-only token injection used during the first drill was replaced the same day by a GitOps-managed secret, `homelab-api-github-actions`, at workloads revision `9852145`. After Argo adopted that secret, a follow-up `POST /rollbacks` returned `202 Accepted`, confirming the durable rollback token path.

### Deploy history timeline includes reason and changelog context

Pass when:

- timeline rows are backed by deployment records
- each row includes deploy reason, compare URL, and result context

Current blocker:

- the current deployments page shows read-only runtime/release history only

## Exit rule

Do not start `E6.1`, `E6.2`, or `E6.6` implementation until this audit is updated and the result is explicitly changed from `NO-GO` to `GO`.
