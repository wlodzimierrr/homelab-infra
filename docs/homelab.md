 Homelab Platform Roadmap (Solo Engineer)

## Executive summary

You should build this platform in **small, production-like slices**: establish a secure GitOps foundation first, then ship a thin vertical slice of your app (frontend + backend + DB + auth), then add observability, and finally evolve into self-service workflows. The primary strategy is:

1. **Stabilize the platform control plane** (cluster, ingress, certs, Argo CD, baseline RBAC).
2. **Deliver one end-to-end workload** (React + FastAPI + Postgres + JWT) through GitOps.
3. **Add guardrails and visibility** (secrets management, policy, monitoring, logs, dashboards).
4. **Automate promotion and developer experience** only after baseline reliability is proven.

For a solo homelab, the key anti-pattern to avoid is building internal platform abstractions before you have repeated pain. Start manual-but-documented, then automate the repeated steps.

---

## Recommended implementation order (opinionated)

1. Phase 0: Foundation hardening and repo strategy
2. Phase 1: Minimum secure GitOps + app vertical slice
3. Phase 2: CI/CD image flow and promotion mechanics
4. Phase 3: Security depth and policy enforcement
5. Phase 4: Observability and operational maturity
6. Phase 5: Self-service and multi-environment expansion

---


## Current-state alignment (based on this repo)

Below is the practical alignment between this roadmap and what appears to already exist in the repository today.

### Already done / materially in place
- **Cluster and networking foundation:** k3s HA install automation, MetalLB, Traefik exposure, and gateway forwarding/WireGuard automation are present.
- **Core platform bootstrap automation:** Argo CD and cert-manager install playbooks/roles exist.
- **Security baseline partially implemented:** Argo CD `admin.enabled=false`, local RBAC defaults, and Argo CD network policy templating are already configured.
- **Infra-as-code for gateway layer:** Terraform for gateway networking/server provisioning exists.

### Partially done (needs completion/validation)
- **Argo CD RBAC hardening:** base policy exists, but project-level boundaries and end-to-end claim mapping still need validation.
- **Network-policy posture:** Argo CD policy exists; app namespace default-deny and explicit allow-lists are not yet represented as a complete standard.
- **Ingress security model:** controls exist in templates/vars, but OAuth/Cloudflare Zero Trust decision and rollout are pending.

### Not yet present (major gaps)
- Application code and manifests for the target platform app (React/FastAPI/Postgres/JWT).
- CI pipelines for build/test/publish + image promotion automation.
- GitOps workloads-repo structure for multi-environment promotion.
- Secrets-as-code workflow (SOPS/Sealed Secrets) and rotation automation.
- Observability stack and runbooks (Prometheus/Grafana/Loki dashboards/alerts).

### Ticket status normalization
Use these tags in tracking:
- **DONE**: implemented in repo and minimally validated.
- **IN PROGRESS**: artifacts exist but missing policy checks/tests/runbook validation.
- **TODO**: no meaningful implementation yet.

Recommended immediate relabeling:
- **DONE:** T1.1.2 (admin disable + baseline RBAC), portions of T0.3.1 (platform namespace policy), and platform install foundations supporting Phase 0.
- **IN PROGRESS:** T3.1.1, T0.3.1 (app namespaces), T1.3.2.
- **TODO:** most of E1.2, all of E2.x, most of E3.2/E3.3, all E4.x and E5.x.

---

## Re-baselined next steps (aligned to what is already done)

1. **Finish Phase 0 closure:** explicitly validate/standardize namespace policies and service-account least privilege.
2. **Accelerate to app vertical slice (E1.2/E1.3):** this is now the highest-value gap.
3. **Immediately follow with CI + image promotion (E2):** avoid manual deployment drift once app exists.
4. **Then complete identity/secrets decisions (E3.2/E3.3):** before opening access broadly.
5. **Add observability (E4) once app traffic exists:** dashboards before portal/self-service.

## Phased roadmap

### Phase 0 (Week 1-2): Platform baseline and delivery model
**Goal:** lock in architecture boundaries and bootstrap conventions before feature work.

#### Epics
- E0.1 Platform architecture and ADR baseline
- E0.2 GitOps repository topology and standards
- E0.3 Cluster baseline hardening

---

### Phase 1 (Week 3-5): First production-like app via GitOps
**Goal:** deploy a complete, secured, observable-enough MVP workload.

#### Epics
- E1.1 App-of-Apps bootstrap in Argo CD
- E1.2 Homelab app MVP (React + FastAPI + Postgres)
- E1.3 Authentication and app access controls

---

### Phase 2 (Week 6-7): CI and image promotion
**Goal:** deterministic build/deploy workflow with low manual toil.

#### Epics
- E2.1 CI pipeline with immutable artifacts
- E2.2 Automated image tag promotion for GitOps
- E2.3 Registry and provenance basics

---

### Phase 3 (Week 8-10): Security hardening and secrets lifecycle
**Goal:** remove obvious security debt and establish policy guardrails.

#### Epics
- E3.1 Argo CD and Kubernetes RBAC enforcement
- E3.2 External auth integration (OAuth or Cloudflare Zero Trust)
- E3.3 Secrets encryption and rotation workflow

---

### Phase 4 (Week 11-13): Observability stack and SLO-lite operations
**Goal:** detect failures quickly and understand deployment health.

#### Epics
- E4.1 Metrics and dashboards (Prometheus + Grafana)
- E4.2 Centralized logging (Loki preferred for homelab simplicity)
- E4.3 Deployment, health, and change visibility

---

### Phase 5 (Week 14+): Platform products and multi-env scale-out
**Goal:** add developer-experience features without breaking simplicity.

#### Epics
- E5.1 Multi-environment GitOps structure (dev/staging/prod)
- E5.2 Self-service project bootstrap
- E5.3 Internal developer portal seed

---

## Epics and detailed tickets

> **Ticket format:** Title, Description, Acceptance Criteria, Dependencies, Complexity, Risk.

### E0.1 Platform architecture and ADR baseline

#### T0.1.1 Create ADR repository structure and template
- **Description:** Add `/docs/adr` with a lightweight ADR template capturing context, decision, alternatives, consequences.
- **Acceptance Criteria:**
  - ADR template exists and is referenced in README.
  - At least 3 ADRs drafted from current decisions.
- **Dependencies:** None
- **Complexity:** S
- **Risk:** Low

#### T0.1.2 Write initial ADR set (cluster distro, GitOps model, secrets strategy placeholder)
- **Description:** Capture current and pending architectural choices to prevent drift.
- **Acceptance Criteria:**
  - ADR-001: K3s + homelab constraints.
  - ADR-002: Argo CD App-of-Apps structure.
  - ADR-003: Secrets approach candidates (SOPS vs Sealed Secrets) with decision date.
- **Dependencies:** T0.1.1
- **Complexity:** S
- **Risk:** Low

### E0.2 GitOps repository topology and standards

#### T0.2.1 Create dedicated workloads repo skeleton
- **Description:** Build a separate repo for app manifests with `bootstrap/`, `platform/`, `apps/`, and environment folders.
- **Acceptance Criteria:**
  - Repo contains documented folder conventions.
  - Argo CD can target repo/paths without ambiguity.
- **Dependencies:** T0.1.2
- **Complexity:** M
- **Risk:** Medium

#### T0.2.2 Define manifest layering strategy
- **Description:** Decide Kustomize overlays (recommended) or Helm value layering for dev/prod.
- **Acceptance Criteria:**
  - One example app demonstrates base + overlay.
  - Promotion path between envs documented.
- **Dependencies:** T0.2.1
- **Complexity:** M
- **Risk:** Medium

### E0.3 Cluster baseline hardening

#### T0.3.1 Enforce namespace and network policy baseline
- **Description:** Add default-deny ingress/egress where feasible and explicit allow rules for platform components.
- **Acceptance Criteria:**
  - Default-deny policy applied to app namespaces.
  - App and Argo CD still function after policy rollout.
- **Dependencies:** T0.2.1
- **Complexity:** M
- **Risk:** Medium

#### T0.3.2 Define service account model for Kubernetes API access
- **Description:** Create minimal RBAC roles for backend API interactions with cluster.
- **Acceptance Criteria:**
  - Dedicated ServiceAccount exists for backend.
  - Role/RoleBinding grant least-privilege verbs/resources.
- **Dependencies:** T0.3.1
- **Complexity:** M
- **Risk:** Medium

---

### E1.1 App-of-Apps bootstrap in Argo CD

#### T1.1.1 Create root app and child app definitions
- **Description:** Implement App-of-Apps with separate child apps for platform and workloads.
- **Acceptance Criteria:**
  - Root app syncs all child apps successfully.
  - Failed child app does not block inspection/recovery.
- **Dependencies:** T0.2.1
- **Complexity:** M
- **Risk:** Medium

#### T1.1.2 Disable Argo CD admin and enforce local RBAC roles
- **Description:** Remove default admin usage and define operator/read-only roles.
- **Acceptance Criteria:**
  - `admin` login disabled.
  - At least two non-admin role mappings validated.
- **Dependencies:** T1.1.1
- **Complexity:** M
- **Risk:** High

### E1.2 Homelab app MVP (React + FastAPI + Postgres)

#### T1.2.1 Scaffold backend API with health, auth, and metadata endpoints
- **Description:** Build FastAPI skeleton with `/health`, `/auth/login`, `/projects` endpoints.
- **Acceptance Criteria:**
  - Endpoint tests pass locally.
  - OpenAPI schema generated and committed.
- **Dependencies:** None
- **Complexity:** M
- **Risk:** Medium

#### T1.2.2 Deploy PostgreSQL with persistent volume and migration workflow
- **Description:** Provision Postgres via Helm/manifests and add schema migrations (Alembic recommended).
- **Status:** DONE (2026-03-02)
- **Acceptance Criteria:**
  - DB survives pod restart.
  - Migration can run idempotently in CI and cluster.
- **Dependencies:** T1.2.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - In-cluster persistence check passed: inserted marker row into `projects`, restarted `homelab-api-postgres-0`, row remained present.
  - Argo CD rollout of hardened migration hook succeeded: `homelab-api-dev` reached `Synced | Healthy` with operation state `Succeeded`.
  - CI migration idempotency check implemented and passing in `apps/portal/.github/workflows/backend-ci.yml` (`python -m alembic upgrade head` executed twice).

#### T1.2.3 Containerize and deploy frontend + backend through GitOps
- **Description:** Build/deploy both services with environment config and ingress routing.
- **Acceptance Criteria:**
  - Frontend can authenticate against backend.
  - Backend can read/write Postgres metadata.
- **Dependencies:** T1.2.1, T1.2.2, T1.1.1
- **Complexity:** L
- **Risk:** High

### E1.3 Authentication and app access controls

#### T1.3.1 Implement JWT issuance and validation path
- **Description:** Add token generation, expiration, refresh strategy, and protected routes.
- **Acceptance Criteria:**
  - Unauthorized access to protected endpoints returns 401.
  - Refresh/token expiry behavior documented and tested.
- **Dependencies:** T1.2.1
- **Complexity:** M
- **Risk:** Medium

#### T1.3.2 Add ingress auth gate (initially basic, evolving to OAuth/ZT)
- **Description:** Protect app entry point with interim controls before SSO integration.
- **Acceptance Criteria:**
  - Unauthenticated ingress requests blocked.
  - Emergency break-glass path documented.
- **Dependencies:** T1.2.3
- **Complexity:** S
- **Risk:** Medium

---

### E2.1 CI pipeline with immutable artifacts

#### T2.1.1 Build/test/publish backend and frontend images in CI
- **Description:** Add pipeline stages for lint/test/build/publish to GHCR or Harbor.
- **Acceptance Criteria:**
  - Every main branch merge publishes immutable image tags.
  - Failed tests block publish.
- **Dependencies:** T1.2.3
- **Complexity:** M
- **Risk:** Medium

#### T2.1.2 Attach SBOM/provenance metadata to images
- **Description:** Generate SBOM and store build provenance to improve auditability.
- **Acceptance Criteria:**
  - CI artifact contains SBOM for each image.
  - Build metadata traceable to commit SHA.
- **Dependencies:** T2.1.1
- **Complexity:** M
- **Risk:** Medium

### E2.2 Automated image tag promotion for GitOps

#### T2.2.1 Implement auto-update controller (Argo CD Image Updater or equivalent)
- **Description:** Wire automated updates of image tags in GitOps manifests for dev.
- **Acceptance Criteria:**
  - New dev image tag triggers Git commit/PR automatically.
  - Sync applies without manual tag edits.
- **Dependencies:** T2.1.1, T0.2.2
- **Complexity:** M
- **Risk:** Medium

#### T2.2.2 Introduce gated promotion to staging/prod
- **Description:** Require manual approval or policy checks before higher env promotion.
- **Acceptance Criteria:**
  - Promotion pipeline supports approve/reject checkpoint.
  - Rollback procedure tested once.
- **Dependencies:** T2.2.1
- **Complexity:** L
- **Risk:** High

### E2.3 Registry and provenance basics

#### T2.3.1 Select and configure primary registry (GHCR default)
- **Description:** Standardize on one registry and auth pattern for cluster pulls.
- **Acceptance Criteria:**
  - Pull secrets configured and rotated procedure documented.
  - Registry retention policy defined.
- **Dependencies:** T2.1.1
- **Complexity:** S
- **Risk:** Low

---

### E3.1 Argo CD and Kubernetes RBAC enforcement

#### T3.1.1 Define Argo CD project-level RBAC boundaries
- **Description:** Restrict source repos, destinations, and namespaces by project.
- **Acceptance Criteria:**
  - App projects cannot deploy outside assigned namespaces.
  - Unauthorized repo path usage denied.
- **Dependencies:** T1.1.2
- **Complexity:** M
- **Risk:** High

#### T3.1.2 Audit and tighten Kubernetes service account permissions
- **Description:** Review all platform service accounts and reduce wildcards.
- **Acceptance Criteria:**
  - No cluster-admin bindings for app workloads.
  - RBAC audit report committed.
- **Dependencies:** T0.3.2
- **Complexity:** M
- **Risk:** Medium

### E3.2 External auth integration

#### T3.2.1 Decide OAuth provider vs Cloudflare Zero Trust
- **Description:** Evaluate operational burden, identity source, and ingress integration.
- **Acceptance Criteria:**
  - Decision ADR accepted.
  - Integration plan includes fallback access method.
- **Dependencies:** T1.3.2
- **Complexity:** S
- **Risk:** Medium

#### T3.2.2 Implement chosen SSO integration for Argo CD and app ingress
- **Description:** Replace interim auth with centralized identity control.
- **Acceptance Criteria:**
  - User/group claims map to Argo CD and app roles.
  - Local admin bypass disabled except break-glass.
- **Dependencies:** T3.2.1
- **Complexity:** L
- **Risk:** High

### E3.3 Secrets encryption and rotation workflow

#### T3.3.1 Implement secrets-as-code with SOPS (recommended default)
- **Description:** Encrypt secret manifests at rest in Git and decrypt in-cluster.
- **Acceptance Criteria:**
  - Secrets in repo are encrypted only.
  - Argo CD can deploy and consume decrypted secrets.
- **Dependencies:** T1.1.1
- **Complexity:** M
- **Risk:** High

#### T3.3.2 Create quarterly secret rotation runbook and automation hooks
- **Description:** Define repeatable process for rotating DB creds, tokens, and registry secrets.
- **Acceptance Criteria:**
  - Rotation checklist exists and is tested once.
  - Rotation causes no prolonged outage (>5 min target).
- **Dependencies:** T3.3.1
- **Complexity:** M
- **Risk:** Medium

---

### E4.1 Metrics and dashboards

#### T4.1.1 Deploy kube-prometheus-stack with minimal retention
- **Description:** Install Prometheus/Grafana tuned for homelab resource limits.
- **Acceptance Criteria:**
  - Cluster/node/pod metrics visible.
  - Retention and storage usage documented.
- **Dependencies:** T1.1.1
- **Complexity:** M
- **Risk:** Medium

#### T4.1.2 Build platform health dashboard
- **Description:** Create dashboards for app availability, deployment status, and error rates.
- **Acceptance Criteria:**
  - Single dashboard shows API health, DB up status, frontend reachability.
  - Alert threshold prototypes configured.
- **Dependencies:** T4.1.1, T1.2.3
- **Complexity:** M
- **Risk:** Medium

### E4.2 Centralized logging

#### T4.2.1 Deploy Loki + Promtail and collect workload logs
- **Description:** Centralize logs with low-ops stack; defer ELK unless scale demands.
- **Acceptance Criteria:**
  - Logs searchable by namespace/app/container.
  - Basic retention policy active.
- **Dependencies:** T1.2.3
- **Complexity:** M
- **Risk:** Medium

#### T4.2.2 Add log-to-triage workflow documentation
- **Description:** Define how to correlate logs with deployments and incidents.
- **Acceptance Criteria:**
  - Runbook includes at least 3 common failure scenarios.
  - Time-to-diagnose baseline measured once.
- **Dependencies:** T4.2.1
- **Complexity:** S
- **Risk:** Low

### E4.3 Deployment and change visibility

#### T4.3.1 Build release dashboard (commits → image tags → Argo sync)
- **Description:** Surface end-to-end delivery state for confidence and debugging.
- **Acceptance Criteria:**
  - Dashboard links commit SHA to deployed image and sync status.
  - Drift detection visible.
- **Dependencies:** T2.2.2, T4.1.1
- **Complexity:** L
- **Risk:** Medium

---

### E5.1 Multi-environment GitOps structure

#### T5.1.1 Split environments with clear promotion contracts
- **Description:** Implement dev/staging/prod overlays and promotion rules.
- **Acceptance Criteria:**
  - Environment differences are declarative and auditable.
  - Promotion between envs uses documented workflow only.
- **Dependencies:** T2.2.2
- **Complexity:** L
- **Risk:** High

### E5.2 Self-service project bootstrap

#### T5.2.1 Create project template generator (repo + manifests + CI seed)
- **Description:** Script/template that scaffolds a new service in <30 minutes.
- **Acceptance Criteria:**
  - New project includes CI, deployment manifests, and baseline policies.
  - At least one generated project deployed successfully.
- **Dependencies:** T5.1.1
- **Complexity:** XL
- **Risk:** High

### E5.3 Internal developer portal seed

#### T5.3.1 Publish a lightweight service catalog page
- **Description:** Start with static/service metadata view before full portal platform.
- **Acceptance Criteria:**
  - Catalog includes owner, repo, runbook, env status links.
  - Data source is Git (not manual spreadsheet).
- **Dependencies:** T5.2.1
- **Complexity:** M
- **Risk:** Medium

---

## ADR candidates (architectural decision points)

1. **ADR: Secrets management approach** (SOPS vs Sealed Secrets vs external secret manager).
2. **ADR: GitOps layout** (single repo vs split repos; environment layering model).
3. **ADR: Manifest tooling** (Kustomize vs Helm-first vs mixed strategy).
4. **ADR: AuthN/AuthZ pattern** (OAuth provider vs Cloudflare Zero Trust; claim mapping).
5. **ADR: Registry standard** (GHCR vs Harbor, retention and vulnerability scanning).
6. **ADR: Observability stack scope** (Loki vs ELK; alerting toolchain depth).
7. **ADR: Promotion model** (auto-dev, gated staging/prod semantics).
8. **ADR: Multi-env branching strategy** (folder-based trunk vs branch-per-env).

---

## Sequencing risks and blockers

### Top blockers
- **Identity integration uncertainty:** Delaying OAuth/ZT decision will stall Argo CD RBAC hardening.
- **Secrets strategy churn:** Migrating from plaintext/adhoc secrets later is painful and risky.
- **GitOps structure indecision:** Reorganizing repo topology midstream creates migration overhead.
- **Over-automation too early:** Building full self-service before stable deploy standards adds rework.

### High-risk sequencing mistakes to avoid
1. Implementing self-service provisioning before deterministic CI/CD and promotion exists.
2. Building multi-env overlays before single-env reliability is proven.
3. Introducing ELK early (resource-heavy) before proving Loki is insufficient.
4. Delaying RBAC and admin disablement until after broad user access is granted.

---

## What NOT to build yet (anti-scope creep)

- Full internal developer portal platform (Backstage-class) before you have >5 services.
- Complex policy engines (OPA/Gatekeeper/Kyverno) until baseline RBAC/network policies are stable.
- Service mesh (Istio/Linkerd) unless you have concrete mTLS/traffic-splitting requirements.
- Multi-cluster federation before one cluster runs reliably for several weeks.
- Custom deployment controllers when Argo CD + Image Updater already solves current needs.

---

## Maturity model (Level 1 → Level 5)

### Level 1: Repeatable manual platform
- Cluster and Argo CD installed.
- App deployed via GitOps manually updated tags.
- Basic ingress auth and backups.

### Level 2: Secure baseline platform
- Argo admin disabled, RBAC enforced.
- JWT auth implemented; service account least privilege applied.
- Secrets encrypted in Git.

### Level 3: Automated delivery platform
- CI builds/tests/publishes immutable images.
- Auto image updates in dev; gated promotion to higher envs.
- Rollback playbook tested.

### Level 4: Observable and operable platform
- Metrics, logs, and release dashboards in place.
- Alerting for key failures.
- Incident/runbook workflows consistently used.

### Level 5: Productized internal platform
- Multi-environment promotion is routine.
- New service bootstrap is semi-self-service.
- Lightweight internal catalog/portal supports discoverability and operations.

---

## Early automation to reduce rework

Prioritize these automations as soon as baseline app deploy works:

1. **Template-driven manifests** (base app chart/overlay skeleton).
2. **CI reusable workflows** (shared build/test/publish jobs).
3. **Automated image bump PRs/commits for dev.**
4. **Secrets encryption/decryption workflow in CI and local tooling.**
5. **One-command environment bootstrap scripts** for new cluster/app namespaces.
6. **Release traceability links** (commit → image → deployment annotation).

These yield compounding benefits and reduce migration pain later.

---

## Suggested timeline (evenings/weekends realistic plan)

Assumption: ~8-12 hours/week.

- **Month 1:** Phase 0 + Phase 1 core (first end-to-end app live through Argo CD).
- **Month 2:** Phase 2 automation (CI, image promotion) + start Phase 3 decisions.
- **Month 3:** Complete Phase 3 security + Phase 4 observability baseline.
- **Month 4+:** Phase 5 only after at least 3-4 weeks of stable operations.

Practical checkpoint rule: do not advance phases unless current phase has at least one "fire drill" (rollback/recovery test) completed successfully.

---

## Architectural evolution diagram (textual)

### Stage A (Foundation)
Developer push → Git (infra/workloads repos) → Argo CD sync → K3s cluster resources.

### Stage B (Vertical slice)
User → Ingress → React frontend → FastAPI backend → PostgreSQL.

### Stage C (Automated delivery)
Developer push → CI build/test → image registry publish → image updater commit/PR → Argo CD sync.

### Stage D (Secure + observable platform)
Identity provider/Cloudflare ZT → ingress and Argo CD auth;
Prometheus/Grafana + Loki ingest telemetry from cluster/app;
Dashboards map deployment state to runtime health.

### Stage E (Platform product)
Project template automation creates repo/manifests/pipelines;
Multi-env promotion workflow governs dev → staging → prod;
Catalog view surfaces ownership, status, and operational links.
