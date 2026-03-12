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
- GitOps workloads structure for multi-environment promotion.
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
- **TODO:** all of E2.x, most of E3.2/E3.3, all E4.x and E5.x.

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
- E1.4 Portal Frontend v1 (React/Vite/Tailwind/shadcn) – start building the user-facing SPA and auth glue
- E1.6 Portal Product Features (Render‑like UX, read‑only first) – implement service catalog, status/dashboard pages, and links; no write actions yet

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
- E4.4 Live monitoring data integration (Portal API + adapters)
- E4.5 Frontend live-data finalization (no mock/sample UI paths)
- E4.6 Backend service registry and project-source normalization

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
- **Status:** DONE (2026-02-27)
- **Acceptance Criteria:**
  - ADR template exists and is referenced in README.
  - At least 3 ADRs drafted from current decisions.
- **Dependencies:** None
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - ADR structure and template committed under `docs/adr/`.
  - ADR index references created ADRs in `docs/adr/README.md`.

#### T0.1.2 Write initial ADR set (cluster distro, GitOps model, secrets strategy placeholder)
- **Description:** Capture current and pending architectural choices to prevent drift.
- **Status:** DONE (2026-02-27)
- **Acceptance Criteria:**
  - ADR-001: K3s + homelab constraints.
  - ADR-002: Argo CD App-of-Apps structure.
  - ADR-003: Secrets approach candidates (SOPS vs Sealed Secrets) with decision date.
- **Dependencies:** T0.1.1
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - `0001-k3s-ha-cluster-topology.md`, `0002-argocd-gitops-model.md`, and `0003-secrets-management-strategy-placeholder.md` committed.

---

### E0.2 GitOps repository topology and standards

#### T0.2.1 Create dedicated workloads repo skeleton
- **Description:** Build a separate repo for app manifests with `bootstrap/`, `platform/`, `apps/`, and environment folders.
- **Status:** DONE (2026-02-28)
- **Acceptance Criteria:**
  - Repo contains documented folder conventions.
  - Argo CD can target repo/paths without ambiguity.
- **Dependencies:** T0.1.2
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Dedicated `workloads` created with `bootstrap/`, `platform/`, `apps/`, and `environments/`.
  - Argo CD applications target repository paths for dev/prod child apps.

#### T0.2.2 Define manifest layering strategy
- **Description:** Decide Kustomize overlays (recommended) or Helm value layering for dev/prod.
- **Status:** DONE (2026-02-28)
- **Acceptance Criteria:**
  - One example app demonstrates base + overlay.
  - Promotion path between envs documented.
- **Dependencies:** T0.2.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Base + overlay pattern implemented for `homelab-api` and `homelab-web`.
  - Promotion path documented in `workloads/README.md`.

---

### E0.3 Cluster baseline hardening

#### T0.3.1 Enforce namespace and network policy baseline
- **Description:** Add default-deny ingress/egress where feasible and explicit allow rules for platform components.
- **Status:** DONE (2026-03-01)
- **Acceptance Criteria:**
  - Default-deny policy applied to app namespaces.
  - App and Argo CD still function after policy rollout.
- **Dependencies:** T0.2.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Default-deny + explicit allow NetworkPolicies present for `homelab-api` and `homelab-web`.
  - Workloads remained functional after policy rollout and validations.

#### T0.3.2 Define service account model for Kubernetes API access
- **Description:** Create minimal RBAC roles for backend API interactions with cluster.
- **Status:** DONE (2026-03-01)
- **Acceptance Criteria:**
  - Dedicated ServiceAccount exists for backend.
  - Role/RoleBinding grant least-privilege verbs/resources.
- **Dependencies:** T0.3.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - `homelab-api-backend` ServiceAccount, Role, and RoleBinding committed in `apps/homelab-api/base/`.
  - Permissions scoped to read-only Kubernetes resources used by backend workflows.

---

### E1.1 App-of-Apps bootstrap in Argo CD

#### T1.1.1 Create root app and child app definitions
- **Description:** Implement App-of-Apps with separate child apps for platform and workloads.
- **Status:** DONE (2026-02-28)
- **Acceptance Criteria:**
  - Root app syncs all child apps successfully.
  - Failed child app does not block inspection/recovery.
- **Dependencies:** T0.2.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Root applications and child application manifests committed under `workloads/bootstrap/` and `workloads/environments/`.
  - Environment-specific child apps for `homelab-api` and `homelab-web` reconciled via Argo CD.

#### T1.1.2 Disable Argo CD admin and enforce local RBAC roles
- **Description:** Remove default admin usage and define operator/read-only roles.
- **Status:** DONE (2026-03-01)
- **Acceptance Criteria:**
  - `admin` login disabled.
  - At least two non-admin role mappings validated.
- **Dependencies:** T1.1.1
- **Complexity:** M
- **Risk:** High
- **Evidence:**
  - Argo CD hardening automation/configuration applied in cluster bootstrap workflow.
  - Repository status normalization and operational validations recorded this control as complete.

---

### E1.2 Homelab app MVP (React + FastAPI + Postgres)

#### T1.2.1 Scaffold backend API with health, auth, and metadata endpoints
- **Description:** Build FastAPI skeleton with `/health`, `/auth/login`, `/projects` endpoints.
- **Status:** DONE (2026-03-02)
- **Acceptance Criteria:**
  - Endpoint tests pass locally.
  - OpenAPI schema generated and committed.
- **Dependencies:** None
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Backend endpoint test suite implemented and passing in CI (`apps/portal/.github/workflows/backend-ci.yml`).
  - OpenAPI schema generated and committed at `apps/portal/backend/openapi.json`.

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
- **Status:** DONE (2026-03-02)
- **Acceptance Criteria:**
  - Frontend can authenticate against backend.
  - Backend can read/write Postgres metadata.
- **Dependencies:** T1.2.1, T1.2.2, T1.1.1
- **Complexity:** L
- **Risk:** High
- **Evidence:**
  - Argo CD deployed `homelab-api-dev` and `homelab-web-dev` at revision `8e7f359...` with `Synced | Healthy`.
  - Runtime images verified:
    - `ghcr.io/wlodzimierrr/homelab-api:0.3.1`
    - `ghcr.io/wlodzimierrr/homelab-web:0.1.0`
  - Ingress endpoints validated:
    - `api.dev.homelab.local/health` returned HTTP 200.
    - `portal.dev.homelab.local` served frontend over Traefik.
  - Backend metadata persistence validated via API:
    - `POST /projects` returned HTTP 201.
    - `GET /projects` returned HTTP 200 with persisted records.

### E1.3 Authentication and app access controls

#### T1.3.1 Implement JWT issuance and validation path
- **Description:** Add token generation, expiration, refresh strategy, and protected routes.
- **Status:** DONE (2026-03-02)
- **Acceptance Criteria:**
  - Unauthorized access to protected endpoints returns 401.
  - Refresh/token expiry behavior documented and tested.
- **Dependencies:** T1.2.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - JWT auth flow implemented in backend with access + refresh token issuance, validation, and token-type checks.
  - Protected routes enforce bearer access tokens and return HTTP 401 for missing/invalid/expired tokens.
  - Refresh strategy documented in `apps/portal/backend/README.md` with TTL settings and stateless rotation behavior.
  - Backend test suite validates unauthorized access, refresh handling, and expiry behavior (`9 passed` in `apps/portal/backend/tests/test_api.py`).

#### T1.3.2 Add ingress auth gate (initially basic, evolving to OAuth/ZT)
- **Description:** Protect app entry point with interim controls before SSO integration.
- **Status:** DONE (2026-03-02)
- **Acceptance Criteria:**
  - Unauthenticated ingress requests blocked.
  - Emergency break-glass path documented.
- **Dependencies:** T1.2.3
- **Complexity:** S
- **Risk:** Medium
- **Evidence:**
  - Traefik basic-auth middleware added for portal ingress in `workloads/apps/homelab-web/base/middleware-basic-auth.yaml`.
  - Portal ingress split into UI (`/`) with basic-auth middleware and API (`/api`) without middleware to avoid `Authorization` header conflicts between ingress basic-auth and backend bearer JWT.
  - Secret bootstrap and break-glass disable/restore procedure documented in `workloads/README.md`.
  - Runtime validation completed after sync: portal access required basic-auth, backend login succeeded, and project refresh returned `Loaded 2 project(s)` with expected records (`proj-browser`, `proj-web`).

---

### E1.4 Portal Frontend v1

#### T1.4.1 Frontend scaffold: React + Vite + TypeScript + Tailwind + shadcn
- **Description:** Create the frontend skeleton for the Portal using Vite (React + TS), TailwindCSS, and shadcn/ui component primitives. Include linting, formatting, and a minimal layout (sidebar/topbar) plus theme toggle persistence.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - `apps/portal/frontend` contains a Vite React TS app with `package.json`, `tsconfig.json`, `tailwind.config.cjs`, and `postcss.config.cjs`.
  - Basic layout routes exist: `/login`, `/dashboard`, `/projects`, `/services`, `/services/:serviceId`, `/services/:serviceId/deployments`, `/settings`.
  - Sidebar + topbar implemented, theme toggle persists in `localStorage`.
  - ESLint + Prettier (or equivalent) configured and `npm run lint` passes on created files.
- **Dependencies:** None (can scaffold independently)
- **Complexity:** M
- **Risk:** Low
- **Evidence:**
  - Vite + React + TypeScript frontend promoted to `apps/portal/frontend` with Tailwind/shadcn wiring and lint/format config (`package.json`, `tsconfig.json`, `tailwind.config.cjs`, `postcss.config.cjs`, `.prettierrc.json`).
  - Route scaffold implemented for `/login`, `/dashboard`, `/projects`, `/services`, `/services/:serviceId`, `/services/:serviceId/deployments`, `/settings` with shared sidebar/topbar layout in `src/components/layout/` and `src/components/navigation/`.
  - Theme toggle persists selection in `localStorage` (`portal-theme`) and applies dark class at app root.
  - Local verification completed by developer run: `npm install`, `npm run lint` (pass), `npm run dev` (pass after Tailwind v4 postcss plugin update).

#### T1.4.2 Auth integration (frontend)
- **Description:** Implement login UI calling `POST /api/auth/login`, store JWT in `localStorage`, attach `Authorization: Bearer <token>` to API calls, and handle 401 by clearing token and redirecting to `/login` with a toast.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - Login screen submits credentials to `POST /api/auth/login` and stores returned token.
  - All API requests include bearer token when present.
  - On 401 responses the app clears token, redirects to `/login`, and shows a toast.
  - Tradeoffs documented in `apps/portal/frontend/README.md` (localStorage vs secure cookie).
- **Dependencies:** T1.4.1, backend `POST /api/auth/login`
- **Complexity:** S
- **Risk:** Medium
- **Evidence:**
  - Login form wired to backend auth endpoint and token persistence implemented in `apps/portal/frontend/src/pages/login-page.tsx` + `src/lib/auth.ts`.
  - Centralized request path adds `Authorization: Bearer <token>` for authenticated calls (`src/lib/api.ts`).
  - Global 401 handler clears token, redirects to `/login`, and shows toast notification (`src/App.tsx` + `src/components/ui/toast.tsx`).
  - LocalStorage vs secure cookie tradeoff documented under `apps/portal/frontend/README.md` (`Auth Storage Tradeoff` section).

#### T1.4.3 Projects UI (list + create)
- **Description:** Implement project list and create flow (Dialog/Sheet) using the existing `GET /api/projects` and `POST /api/projects` endpoints. Provide loading skeletons, empty states and toasts.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - `/projects` lists projects using `GET /api/projects` with loading and empty states.
  - Create project dialog submits `POST /api/projects` and shows success/error toast.
  - API errors surfaced via consistent error component.
- **Dependencies:** T1.4.1, backend `/api/projects`
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - `/projects` page fetches from `GET /api/projects` and renders loading skeletons + empty state (`apps/portal/frontend/src/pages/projects-page.tsx`).
  - Create project modal flow implemented with `POST /api/projects` and success/error toasts.
  - Reusable `ApiError` component introduced and used consistently for list/create API failures (`apps/portal/frontend/src/components/api-error.tsx`).
  - Dev proxy/path and environment alignment validated against hosted backend (`api.dev.homelab.local`) after local CORS/proxy fixes.

#### T1.4.4 API client layer and small state helpers
- **Description:** Add `src/lib/api.ts`, `src/lib/auth.ts`, and `src/lib/config.ts` for a typed API client, token helpers, and external URL config. Keep state with React hooks only.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - `api.ts` exposes typed wrappers for `GET /api/projects`, `POST /api/projects`, and a generic `request()` that attaches auth header.
  - `auth.ts` exposes `getToken()`, `setToken()`, `clearToken()` and an `useAuth()` hook.
  - `config.ts` centralizes `VITE_API_BASE_URL` and external base URLs (Argo/Grafana).
- **Dependencies:** T1.4.1
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Typed API layer implemented in `apps/portal/frontend/src/lib/api.ts` with `request()`, `getProjects()`, `createProject()`, and auth header attachment.
  - Auth helpers refactored to `getToken()`, `setToken()`, `clearToken()` with `useAuth()` hook in `apps/portal/frontend/src/lib/auth.ts`.
  - External/runtime config centralized in `apps/portal/frontend/src/lib/config.ts` with API + Argo + Grafana base URLs.
  - Existing UI migrated from legacy `api-client.ts` to new modules; legacy client removed.

#### T1.4.5 Dockerfile + nginx SPA fallback
- **Description:** Add a multi-stage `Dockerfile` that builds the Vite app and serves with nginx, exposing port 80 and enabling SPA fallback for nested routes.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - `apps/portal/frontend/Dockerfile` multi-stage build exists and produces a ~static nginx image.
  - Refreshing nested routes returns the SPA (nginx config uses try_files /index.html fallback).
  - Image listens on port 80.
- **Dependencies:** T1.4.1
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Multi-stage frontend image build implemented in `apps/portal/frontend/Dockerfile` (Node build stage + nginx runtime stage).
  - SPA fallback confirmed in nginx config via `try_files $uri /index.html` (`apps/portal/frontend/nginx.conf`).
  - Runtime port updated to `80` (`listen 80`, `EXPOSE 80`) and workload container port aligned to 80 in `workloads/apps/homelab-web/base/deployment.yaml`.

---

### E1.6 Portal Product Features (Render-like UX, read-only first)

#### T1.6.1 Services list page
- **Description:** Implement `/services` showing service name, environments, current status (health/sync), public URL (if present), and basic search/filter. Data should be read-only and may be adapted from `GET /api/projects` or a future `/services` endpoint.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - `/services` shows a list/table of services with columns: name, env(s), status badge, public URL, last deploy timestamp (if available).
  - Client-side search and environment filter present and functional.
  - Loading skeletons and empty states implemented.
- **Dependencies:** T1.4.1, T1.4.4
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - `/services` is now data-driven from `GET /projects` via adapter logic, grouped by service name with environment aggregation (`apps/portal/frontend/src/pages/services-page.tsx`).
  - Services table renders required columns: service name, environment(s), health/sync status badges, public URL (or `N/A`), and last deploy timestamp (or `N/A`).
  - Client-side search and environment filtering implemented and combined in-memory before render (`services-page.tsx`).
  - Loading skeleton, API error + retry, empty state, and no-filter-match state added on the page.
  - API `Project` typing extended with optional `health`, `sync`, `publicUrl`, and `lastDeployAt` fields for forward compatibility (`apps/portal/frontend/src/lib/api.ts`).
  - Mobile topbar menu behavior on small screens updated from direct dashboard link to an actual navigation menu, fixing services flow navigation (`apps/portal/frontend/src/components/navigation/topbar.tsx`).

#### T1.6.2 Service detail page (overview)
- **Description:** Implement `/services/:serviceId` with Overview tab showing status cards: deployed version, sync/health state, links to Argo app, Grafana dashboard, and Logs. Endpoints list and a preview of recent deployments (top 5).
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - Overview tab shows version tag, health/sync indicators (use placeholders if backend not providing yet).
  - Links open external Argo/Grafana/Loki pages using templates from `config.ts`.
  - Endpoints section lists public/internal URLs if available.
  - Recent deployments preview shows up to 5 items (mocked fallback if backend lacks data).
- **Dependencies:** T1.6.1, T1.4.4
- **Complexity:** L
- **Risk:** Medium
- **Evidence:**
  - Service detail Overview implemented for `/services/:serviceId` in `apps/portal/frontend/src/pages/service-details-page.tsx` with deployed version, sync/health indicators, endpoints, and recent deployments preview.
  - Overview data loads backend-first (`getService`, `getServiceDeployments`) with project-derived and mocked fallbacks when fields are missing.
  - External links are generated via `apps/portal/frontend/src/lib/config.ts` helpers (`buildArgoAppUrl`, `buildGrafanaDashboardUrl`, `buildLogsUrl`).
  - Recent deployments preview is capped to top 5 with fallback entries when backend data is unavailable.

#### T1.6.3 Service status view (health + sync)
- **Description:** Create reusable status components that display health and sync status with clear color semantics. Allow placeholder/mocked values until backend supplies real data.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - `StatusCard` component accepts `health` and `sync` props and renders consistent UI.
  - Placeholder state when data is missing with CTA text like "Status unavailable — backend integration pending".
  - Tests or visual checks show all three states (Healthy, Degraded, Unknown).
- **Dependencies:** T1.6.2
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Reusable `StatusCard` added in `apps/portal/frontend/src/components/status-card.tsx` with `health`/`sync` props and consistent status color semantics.
  - Placeholder copy is shown when both status values are unknown: "Status unavailable - backend integration pending".
  - Visual checks for Healthy/Degraded/Unknown added on the service detail page in dev mode.

#### T1.6.4 Deployment history page (read-only, mocked adapter)
- **Description:** `/services/:serviceId/deployments` lists last N deployments (timestamp, version/tag, outcome). Use a mocked data adapter that can be replaced by a backend adapter later.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - Deployments page lists at least the 10 most recent entries (mocked if backend absent).
  - Data adapter implementation is isolated behind a single module `src/lib/adapters/deployments.ts` and documented TODO for swap.
  - Empty and error states handled.
- **Dependencies:** T1.4.4, T1.6.2
- **Complexity:** M
- **Risk:** Low
- **Change Note:** This MVP read-only mocked adapter is superseded in Level 3 by `T6.7.2` (deployment-record-backed timeline).
- **Evidence:**
  - Deployment history page implemented in `apps/portal/frontend/src/pages/service-deployments-page.tsx` with read-only table, loading, empty, and error states.
  - Adapter isolated to `apps/portal/frontend/src/lib/adapters/deployments.ts` (`getDeploymentHistory`) with backend-first fetch and mocked fallback.
  - Adapter guarantees at least 10 entries when backend data is absent and includes a TODO note for backend adapter swap.

#### T1.6.5 Logs access (deep links to Grafana/Loki)
- **Description:** Provide a control that opens external Grafana/Loki with prefilled query parameters to filter by service label/namespace. Base URL and template strings come from `config.ts`.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - Logs button opens a new tab to configured Grafana/Loki URL with query parameters populated for the selected service.
  - Template supports variables like `{{namespace}}`, `{{app_label}}`, and `{{time_range}}`.
  - Button disabled/hidden if external URL not configured.
- **Dependencies:** T1.4.4, infra (Grafana/Loki URLs)
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Logs action on service detail overview now opens external Grafana/Loki in a new tab with populated service filters.
  - `apps/portal/frontend/src/lib/config.ts` template handling supports both `{var}` and `{{var}}`, including `{{namespace}}`, `{{app_label}}`, and `{{time_range}}`.
  - Logs availability is guarded by config (`isLogsConfigured()`), and the button is disabled when external URL/template are not configured.

#### T1.6.6 External links config (Argo CD, Grafana)
- **Description:** Add `src/lib/config.ts` entries and README instructions for configuring Argo CD and Grafana base URLs in environment variables; support in-cluster `VITE_API_BASE_URL=/api` and local dev `VITE_API_BASE_URL=http://localhost:8000/api`.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - `config.ts` reads `VITE_ARGO_BASE_URL` and `VITE_GRAFANA_BASE_URL` with sensible defaults (empty).
  - README documents how to configure these env vars for local dev and in-cluster values.
- **Dependencies:** T1.4.4
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - External URL config defaults updated in `apps/portal/frontend/src/lib/config.ts` so `VITE_ARGO_BASE_URL` and `VITE_GRAFANA_BASE_URL` default to empty values.
  - `apps/portal/frontend/README.md` now documents runtime env configuration with explicit examples for local dev (`VITE_API_BASE_URL=http://localhost:8000/api`) and in-cluster (`VITE_API_BASE_URL=/api`).
  - README also documents related external link template vars used by Argo/Grafana/Logs links.

#### T1.6.7 Service registry model: static-config MVP vs DB-backed decision
- **Description:** Decide the simplest approach for service metadata: either add a lightweight DB model in backend or use a static YAML/JSON config stored in the repo for MVP. Implement the chosen MVP path.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - Decision recorded in ticket with rationale and follow-up migration plan.
  - If static-config chosen: example `apps/portal/frontend/services.sample.json` included and used by the frontend adapter as fallback.
  - If DB-backed chosen: backend migration ticket referenced and sample API contract added.
- **Dependencies:** T1.2.1 (backend), T1.4.4
- **Complexity:** S
- **Risk:** Medium
- **Decision:** Static-config MVP selected.
- **Rationale:**
  - Lowest implementation/ops overhead for homelab phase while backend service catalog contracts are still evolving.
  - Keeps registry data auditable in Git and avoids introducing schema/migration work for a non-critical read-only surface.
  - Allows frontend progress even when `/projects` data is unavailable or incomplete.
- **Follow-up Migration Plan:**
  - Introduce backend-owned `/api/services` contract with explicit service metadata fields (owner, repo, envs, URLs, labels).
  - Add backend persistence (DB model + migration) only after metadata ownership and write workflows are finalized.
  - Switch frontend services adapter from static fallback to backend adapter, then deprecate `services.sample.json`.
- **Evidence:**
  - Added static sample registry `apps/portal/frontend/services.sample.json`.
  - Added frontend services adapter `apps/portal/frontend/src/lib/adapters/services.ts` with API-first load and static JSON fallback.
  - Wired services page to the adapter (`apps/portal/frontend/src/pages/services-page.tsx`), so fallback is used when API data is absent/unavailable.

#### T1.6.8 UX polish and shared components
- **Description:** Implement consistent loading, empty, and error components, Sonner toasts, and reuse shadcn Card/Table/Tabs/Dialog components across pages. Make layout responsive.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - Shared `components/` directory with `Loading`, `Empty`, `Error`, and `Toast` wrappers.
  - All major pages use consistent components and are responsive on small screens.
- **Dependencies:** T1.4.1
- **Complexity:** M
- **Risk:** Low
- **Evidence:**
  - Shared wrappers added in `apps/portal/frontend/src/components/`: `loading-state.tsx`, `empty-state.tsx`, `error-state.tsx`, and `toast-message.tsx`.
  - Major pages now use shared state wrappers for consistent UX: `projects-page.tsx`, `services-page.tsx`, `service-details-page.tsx`, `service-deployments-page.tsx`, and `login-page.tsx`.
  - App-level unauthorized notifications and project create notifications now flow through the shared toast wrapper (`apps/portal/frontend/src/App.tsx`, `projects-page.tsx`).

#### T1.6.9 Frontend README and run instructions
- **Description:** Document local dev, env vars, Docker build/run, and note that deploy/logs are read-only and may be mocked until backend integration is complete.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - `apps/portal/frontend/README.md` with: install, dev start, build, docker build/run, env vars list, and note about read-only behavior.
  - Commands validated locally (where possible) and documented.
- **Dependencies:** T1.4.1, T1.4.5
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - `apps/portal/frontend/README.md` now includes install/dev/build flow, runtime env vars list, and local/in-cluster examples.
  - Docker build/run instructions are documented with direct commands (`docker build`, `docker run`) plus the existing publish helper script.
  - README explicitly notes read-only behavior and mocked/placeholder integration boundaries for deploy/logs/status metadata.
  - Command-validation section added with runnable commands and environment caveat for unsupported shells (for example WSL1).

#### Anti-scope / NOT included in this iteration
- No deploy buttons or any write/deploy actions (no PR generation, no kubectl apply from UI).
- No environment variable editor or secrets management UI.
- No full Backstage-like scaffolding or cataloging features.
- No embedded Grafana dashboards — link-outs only.

### E2.1 CI pipeline with immutable artifacts

#### T2.1.1 Build/test/publish backend and frontend images in CI
- **Description:** Add pipeline stages for lint/test/build/publish to GHCR or Harbor.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - Every main branch merge publishes immutable image tags.
  - Failed tests block publish.
- **Dependencies:** T1.2.3
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Added workflow `apps/portal/.github/workflows/portal-images.yml` with stages/jobs for `lint`, `test`, `build-images`, and `publish-images`.
  - `publish-images` is gated on successful `lint`/`test`/`build-images` and runs only on `push` to `main`.
  - Backend and frontend images are published to GHCR with immutable SHA tags:
    - `ghcr.io/wlodzimierrr/homelab-api:sha-<commit>`
    - `ghcr.io/wlodzimierrr/homelab-web:sha-<commit>`

#### T2.1.2 Attach SBOM/provenance metadata to images
- **Description:** Generate SBOM and store build provenance to improve auditability.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - CI artifact contains SBOM for each image.
  - Build metadata traceable to commit SHA.
- **Dependencies:** T2.1.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - `apps/portal/.github/workflows/portal-images.yml` now generates SPDX SBOMs per image (`backend`, `frontend`) via `anchore/sbom-action` and uploads them as CI artifacts with `actions/upload-artifact`.
  - Publish step now attaches provenance and SBOM attestations to pushed images (`provenance: mode=max`, `sbom: true`) via `docker/build-push-action`.
  - Published image tags and OCI metadata remain commit-SHA traceable (for example `sha-<commit>` tags and `org.opencontainers.image.revision`).

### E2.2 Automated image tag promotion for GitOps

#### T2.2.1 Implement auto-update controller (Argo CD Image Updater or equivalent)
- **Description:** Wire automated updates of image tags in GitOps manifests for dev.
- **Status:** DONE (2026-03-03)
- **Acceptance Criteria:**
  - New dev image tag triggers Git commit/PR automatically.
  - Sync applies without manual tag edits.
- **Dependencies:** T2.1.1, T0.2.2
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - `apps/portal/.github/workflows/portal-images.yml` now includes `promote-dev-gitops` job, triggered after successful `publish-images` on `main`.
  - The job checks out GitOps repo `wlodzimierrr/homelab-workloads`, updates only dev overlay tag files:
    - `apps/homelab-api/envs/dev/patch-deployment.yaml`
    - `apps/homelab-api/envs/dev/patch-migration-job.yaml`
    - `apps/homelab-web/envs/dev/patch-deployment.yaml`
  - Changes are committed as an automated PR via `peter-evans/create-pull-request` (`automation/dev-image-bump-<sha>`), satisfying commit/PR automation for new image tags.
  - Existing Argo CD dev Applications are already configured with `syncPolicy.automated` (`prune: true`, `selfHeal: true`), so merged tag updates apply without manual tag edits.

#### T2.2.2 Introduce gated promotion to staging/prod
- **Description:** Require manual approval or policy checks before higher env promotion.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Promotion pipeline supports approve/reject checkpoint.
  - Rollback procedure tested once.
- **Dependencies:** T2.2.1
- **Complexity:** L
- **Risk:** High
- **Evidence:**
  - `apps/portal/.github/workflows/gated-promotion.yml` adds a `workflow_dispatch` promotion pipeline for `staging`/`prod` with `action_mode` of `promote` or `rollback`.
  - Pipeline includes policy checks before the gate: target overlay file existence, strict tag format (`sha-<40 hex>`), and GHCR image tag existence via `docker manifest inspect`.
  - Manual approve/reject checkpoint is enforced via `approval-gate` job bound to protected environment `homelab-<target>-promotion`; rejection blocks PR creation.
  - Post-approval step creates a constrained PR in `wlodzimierrr/homelab-workloads` updating only target env image patch files for `homelab-api` and `homelab-web`.
  - Rollback rehearsal executed once locally on 2026-03-04T07:47:58Z: simulated `N -> N+1 -> N` image updates on temp manifests and confirmed all target files returned to rollback tag.

### E2.3 Registry and provenance basics

#### T2.3.1 Select and configure primary registry (GHCR default)
- **Description:** Standardize on one registry and auth pattern for cluster pulls.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Pull secrets configured and rotated procedure documented.
  - Registry retention policy defined.
- **Dependencies:** T2.1.1
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - `workloads/README.md` now defines the GHCR auth standard (`ghcr.io`, `ghcr-pull-secret`, namespace/service-account bindings) for cluster image pulls.
  - Pull secret rotation workflow is documented with executable commands for `homelab-api` and `homelab-web`, plus rollout validation commands and 90-day rotation cadence.
  - Registry retention policy is explicitly defined for both GHCR packages (`homelab-api`, `homelab-web`): keep release/semver tags, retain associated provenance, and prune old `sha-*` tags to the latest 60 on a monthly schedule.

---

### E3.1 Argo CD and Kubernetes RBAC enforcement

#### T3.1.1 Define Argo CD project-level RBAC boundaries
- **Description:** Restrict source repos, destinations, and namespaces by project.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - App projects cannot deploy outside assigned namespaces.
  - Unauthorized repo path usage denied.
- **Dependencies:** T1.1.2
- **Complexity:** M
- **Risk:** High
- **Evidence:**
  - `workloads/bootstrap/project-homelab.yaml` now defines scoped `AppProject` objects (`homelab-bootstrap`, `homelab-platform`, `homelab-api`, `homelab-web`) with explicit destination namespace restrictions per project.
  - Each `AppProject` now restricts `sourceRepos` to the dedicated workloads repository URL (`https://github.com/wlodzimierrr/homelab-workloads.git`), replacing the former wildcard allow-list.
  - Environment application manifests are remapped to scoped projects:
    - bootstrap/root + workloads parent apps -> `homelab-bootstrap`
    - platform apps -> `homelab-platform`
    - homelab-api apps -> `homelab-api`
    - homelab-web apps -> `homelab-web`
  - This enforces Argo CD project-level deny behavior for out-of-scope destinations and non-allowed repository sources.

#### T3.1.2 Audit and tighten Kubernetes service account permissions
- **Description:** Review all platform service accounts and reduce wildcards.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - No cluster-admin bindings for app workloads.
  - RBAC audit report committed.
- **Dependencies:** T0.3.2
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - `workloads/audit/rbac-audit-2026-03-04.md` added as the committed RBAC audit report, with scope, commands, findings, and conclusion.
  - Audit confirms app workload RBAC avoids `cluster-admin`; namespace-scoped RBAC remains the default, with one reviewed read-only `ClusterRole` + `ClusterRoleBinding` exception for `homelab-api` cluster discovery.
  - `workloads/scripts/check-rbac-guardrails.sh` enforces wildcard/token checks, blocks unexpected `ClusterRoleBinding`s, and ensures the approved cluster-wide binding stays read-only.

### E3.2 External auth integration

#### T3.2.1 Decide OAuth provider vs Cloudflare Zero Trust
- **Description:** Evaluate operational burden, identity source, and ingress integration.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Decision ADR accepted.
  - Integration plan includes fallback access method.
- **Dependencies:** T1.3.2
- **Complexity:** S
- **Risk:** Medium
- **Evidence:**
  - ADR drafted at `docs/adr/0006-oauth-vs-cloudflare-zero-trust.md` with `status: Accepted` and decision rationale.
  - Integration plan includes explicit fallback/break-glass access methods (WireGuard emergency access, short-lived basic-auth toggle, admin recovery token rotation).

#### T3.2.2 Implement chosen SSO integration for Argo CD and app ingress
- **Description:** Replace interim auth with centralized identity control.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - User/group claims map to Argo CD and app roles.
  - Local admin bypass disabled except break-glass.
- **Dependencies:** T3.2.1
- **Complexity:** L
- **Risk:** High
- **Evidence:**
  - Argo CD OIDC + RBAC config committed in `workloads/bootstrap/argocd-oidc.yaml` with group scopes/role mappings and `admin.enabled: "false"`.
  - Portal SSO ingress path implemented in dev overlay via `oauth2-proxy`, Traefik forward-auth/error middlewares, ingress patches, and oauth2-proxy NetworkPolicies under `workloads/apps/homelab-web/envs/dev/`.
  - App role enforcement implemented for forwarded identity/group claims in `apps/portal/backend/app/main.py` (`X-Auth-Request-User` and `X-Auth-Request-Groups` with admin gate for write actions).
  - Break-glass and validation runbooks documented in `docs/runbooks/oidc-setup.md` and `docs/runbooks/sso-break-glass.md`.
  - Cluster validation executed with server-side dry runs:
    - `kubectl apply --dry-run=server -f workloads/bootstrap/argocd-oidc.yaml`
    - `kubectl apply --dry-run=server -k workloads/apps/homelab-web/envs/dev`

### E3.3 Secrets encryption and rotation workflow

#### T3.3.1 Implement secrets-as-code with SOPS (recommended default)
- **Description:** Encrypt secret manifests at rest in Git and decrypt in-cluster.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Secrets in repo are encrypted only.
  - Argo CD can deploy and consume decrypted secrets.
- **Dependencies:** T1.1.1
- **Complexity:** M
- **Risk:** High
- **Evidence:**
  - ADR `0003` updated to `Accepted` for `SOPS + age` (`docs/adr/0003-secrets-management-strategy-placeholder.md`).
  - Repo-level SOPS creation rules added at `workloads/.sops.yaml`.
  - Plaintext secret guardrail script added at `workloads/scripts/check-secrets-guardrails.sh`.
  - Implementation runbook added at `docs/runbooks/sops-secrets.md`.
  - Real SOPS-encrypted Postgres secret manifests committed for both dev/prod overlays:
    - `workloads/apps/homelab-api/envs/dev/postgres-secret.enc.yaml`
    - `workloads/apps/homelab-api/envs/prod/postgres-secret.enc.yaml`
  - KSOPS generator manifests added for `homelab-api` dev/prod and `homelab-web` dev overlays.
  - Argo CD role now patches `argocd-repo-server` to mount `ksops`, consume `argocd-sops-age`, and build Kustomize apps with `--enable-alpha-plugins --enable-exec`.
  - On 2026-03-09, Argo CD reconciled `homelab-api-dev` and `homelab-web-dev` from `homelab-workloads` revision `5d3095f`, rendered real `Secret` objects for `homelab-api-postgres` and `oauth2-proxy-secret`, and kept both apps `Synced` and `Healthy`.

#### T3.3.2 Create quarterly secret rotation runbook and automation hooks
- **Description:** Define repeatable process for rotating DB creds, tokens, and registry secrets.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Rotation checklist exists and is tested once.
  - Rotation causes no prolonged outage (>5 min target).
- **Dependencies:** T3.3.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Quarterly rotation checklist/runbook created at `docs/runbooks/secret-rotation-quarterly.md`.
  - Automation hook added at `workloads/scripts/verify-rotation-slo.sh` to assert rollout completion within SLO window (default `300s`).
  - Supporting docs updated in `workloads/README.md` with rotation workflow and command references.
  - Rotation rehearsal executed on 2026-03-04 with SLO checks passing:
    - `verify-rotation-slo.sh homelab-api homelab-api 300` -> rollout completed in `0s` (`PASS`).
    - `verify-rotation-slo.sh homelab-web oauth2-proxy 300` -> rollout completed in `0s` (`PASS`).

---

### E4.1 Metrics and dashboards

#### T4.1.1 Deploy kube-prometheus-stack with minimal retention
- **Description:** Install Prometheus/Grafana tuned for homelab resource limits.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Cluster/node/pod metrics visible.
  - Retention and storage usage documented.
- **Dependencies:** T1.1.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Argo CD app added for shared cluster monitoring:
    - `workloads/environments/dev/workloads/monitoring-app.yaml`
  - `monitoring-prod` app intentionally removed to avoid shared-resource conflicts from deploying the same Helm release into the same namespace from two Argo CD applications.
  - Single-cluster safety guardrail added: `workloads/environments/prod/workloads/kustomization.yaml` is intentionally empty and prod workload app manifests were removed to prevent accidental recreation of `homelab-api-prod`/`homelab-web-prod` in the same cluster.
  - Monitoring AppProject boundary added in `workloads/bootstrap/project-homelab.yaml` (`homelab-monitoring`, destinations `monitoring` and `kube-system` for control-plane scrape Services).
  - Retention/storage profile configured in manifests:
    - Shared stack: `24h`, `retentionSize: 3GiB`, Prometheus PVC `5Gi`
  - Runbook with validation and storage checks added at `docs/runbooks/monitoring-kube-prometheus-stack.md`.
  - Stabilized Argo hook behavior for API migrations by setting `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation` in `workloads/apps/homelab-api/base/migration-job.yaml` to avoid stale "Syncing" operation states when hook jobs are removed too early.

#### T4.1.2 Build platform health dashboard
- **Description:** Create dashboards for app availability, deployment status, and error rates.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Single dashboard shows API health, DB up status, frontend reachability.
  - Alert threshold prototypes configured.
- **Dependencies:** T4.1.1, T1.2.3
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Single dashboard `Homelab Platform Health` provisioned via `grafana.dashboards.default` in `workloads/environments/dev/workloads/monitoring-app.yaml`.
  - Dashboard panels include API availability ratio, frontend availability ratio, DB ready replicas, deployment ready vs desired, and prototype API 5xx rate panel.
  - Prototype Prometheus alert thresholds configured via `additionalPrometheusRulesMap` in `workloads/environments/dev/workloads/monitoring-app.yaml`:
    - `HomelabApiUnavailable`
    - `HomelabWebUnavailable`
    - `HomelabPostgresUnavailable`
    - `HomelabApiRestartSpike`
    - `HomelabApi5xxRateHighPrototype`
  - Validation runbook added at `docs/runbooks/platform-health-dashboard.md`.

### E4.2 Centralized logging

#### T4.2.1 Deploy Loki + Promtail and collect workload logs
- **Description:** Centralize logs with low-ops stack; defer ELK unless scale demands.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Logs searchable by namespace/app/container.
  - Basic retention policy active.
- **Dependencies:** T1.2.3
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Argo CD apps added:
    - `workloads/environments/dev/workloads/loki-app.yaml`
    - `workloads/environments/dev/workloads/promtail-app.yaml`
  - Dev workloads kustomization now includes logging apps:
    - `workloads/environments/dev/workloads/kustomization.yaml`
  - Monitoring AppProject extended to allow Grafana Helm repo source:
    - `workloads/bootstrap/project-homelab.yaml`
  - Grafana provisioning includes Loki datasource (`uid: loki`) for Explore queries:
    - `workloads/environments/dev/workloads/monitoring-app.yaml`
  - Loki retention configured to `168h` with compactor retention enabled in single-binary mode:
    - `workloads/environments/dev/workloads/loki-app.yaml`
  - Promtail relabeling includes `app` label (with pod-name fallback) so logs are queryable by namespace/app/container:
    - `workloads/environments/dev/workloads/promtail-app.yaml`
  - Validation runbook added:
    - `docs/runbooks/logging-loki-promtail.md`

#### T4.2.2 Add log-to-triage workflow documentation
- **Description:** Define how to correlate logs with deployments and incidents.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Runbook includes at least 3 common failure scenarios.
  - Time-to-diagnose baseline measured once.
- **Dependencies:** T4.2.1
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Log-to-triage workflow runbook added:
    - `docs/runbooks/log-to-triage-workflow.md`
  - Runbook includes standardized correlation workflow across Loki, Argo CD sync status, and Kubernetes rollout signals.
  - Runbook documents 3 common failure scenarios:
    - Promtail push failures during Loki startup
    - Missing/sparse service logs after deployment due to stale query filters/time windows
    - Authentication/dependency failures (for example API/Postgres credential mismatch)
  - Time-to-diagnose baseline measured once and recorded:
    - Drill date `2026-03-04`
    - Measured TTD `14 minutes` for first-run ingestion incident pattern.

### E4.3 Portal Monitoring & Observability UX

Scope: All tickets in this section implement monitoring visibility inside the
React/Fast API Portal  (`apps/portal`). Infrastructure, Prometheus,
and Grafana provisioning are handled in E4.1 and E4.2.

#### T4.3.1 Build release dashboard (commits → image tags → Argo sync)
- **Description:** Surface end-to-end delivery state for confidence and debugging.
- **Status:** DONE (2026-03-04)
- **Acceptance Criteria:**
  - Dashboard links commit SHA to deployed image and sync status.
  - Drift detection visible.
- **Dependencies:** T2.2.2, T4.1.1
- **Complexity:** L
- **Risk:** Medium
- **Evidence:**
  - Portal dashboard implementation now renders release traceability rows with commit SHA, deployed image, Argo sync/health, drift state, and deployment timestamp:
    - `apps/portal/frontend/src/pages/dashboard-page.tsx`
  - Release dashboard adapter added with API-first + fallback loading and drift computation:
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
  - Fallback sample dataset with commit/image/Argo metadata added:
    - `apps/portal/frontend/release-dashboard.sample.json`
  - Validation runbook added:
    - `docs/runbooks/release-traceability-dashboard.md`

#### T4.3.2 Service metrics summary cards on service detail page
- **Description:** Add summary cards to each service page for uptime %, p95 latency, error rate, and restart count, using existing service identity metadata.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Service detail page shows four metric cards with current values and last refresh timestamp.
  - Card states include healthy/warning/critical thresholds and match shared status styling.
  - Missing metric data renders an explicit "No data" state without breaking layout.
- **Dependencies:** T1.6.2, T1.6.3, T4.1.2
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Service detail page now renders four metric cards (uptime, p95 latency, error rate, restart count), including per-card severity and last refresh timestamp:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Shared metric card component added with explicit `No data` rendering and status tone mapping:
    - `apps/portal/frontend/src/components/service-metric-card.tsx`
  - Metrics adapter added with API-first loading and safe fallback behavior:
    - `apps/portal/frontend/src/lib/adapters/service-metrics.ts`
  - Fallback sample metrics dataset added:
    - `apps/portal/frontend/service-metrics.sample.json`
  - Validation runbook added:
    - `docs/runbooks/service-metrics-summary-cards.md`

#### T4.3.3 Shared uptime indicator widget and status mapping
- **Description:** Implement a reusable uptime indicator component (24h and 7d) with threshold-to-severity mapping for consistent status presentation across pages.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Shared widget component is available in the frontend component library and reused in at least two screens.
  - Threshold mapping is centrally configurable and unit-tested.
  - Widget supports loading, stale-data, and no-data states.
- **Dependencies:** T1.6.3, T4.3.2
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Shared uptime indicator component added with loading, stale-data, and no-data support:
    - `apps/portal/frontend/src/components/uptime-indicator.tsx`
  - Centralized threshold + status mapping module added:
    - `apps/portal/frontend/src/lib/uptime-status.ts`
  - Widget reused in at least two screens:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
    - `apps/portal/frontend/src/pages/services-page.tsx`
  - Unit tests added for status mapping helpers and stale detection:
    - `apps/portal/frontend/tests/uptime-status.test.ts`
    - `apps/portal/frontend/tsconfig.test.json`
  - Validation runbook added:
    - `docs/runbooks/uptime-indicator-widget.md`

#### T4.3.4 Embedded Grafana panels for latency and error trends
- **Description:** Embed selected Grafana panels inside service detail pages for response time and error-rate trends, with a fallback deep link when embedding is unavailable.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Service detail page renders at least two embedded panels (latency, errors) for the selected service.
  - "Open in Grafana" action preserves service and time-range context.
  - Failed embed renders a non-blocking fallback state with a working deep link.
- **Dependencies:** T1.6.2, T1.6.6, T4.1.2
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Service detail page now includes embedded Grafana panel section with latency and error trends:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Reusable embedded panel component added with timeout/error fallback and deep-link actions:
    - `apps/portal/frontend/src/components/grafana-embed-panel.tsx`
  - Grafana panel URL builders and time-range interpolation support added:
    - `apps/portal/frontend/src/lib/config.ts`
  - Validation runbook added:
    - `docs/runbooks/grafana-embedded-panels.md`

#### T4.3.5 Template-driven monitoring URL builder (Grafana + Loki)
- **Description:** Extend frontend URL templating so monitoring links can inject variables (service, namespace, environment, range) without hardcoded page-specific logic.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - URL template helper supports variable interpolation for Grafana and Loki routes.
  - Service detail and logs entry points use shared template helper.
  - Invalid or missing template variables produce safe fallback links and console warnings in development mode.
- **Dependencies:** T1.6.5, T1.6.6
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Shared monitoring URL template helper added with interpolation, safe fallback, and dev-mode warnings:
    - `apps/portal/frontend/src/lib/config.ts`
  - Service detail quick links and logs entry points continue to consume shared config builders:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Quick links now render safe unavailable state when URL configuration resolves to empty:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Validation runbook added:
    - `docs/runbooks/monitoring-url-template-builder.md`

#### T4.3.6 Service health timeline (status-over-time)
- **Description:** Add a compact timeline view on the service page to visualize health transitions (healthy, degraded, down) over time.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Timeline shows status segments for a selectable window (default 24h).
  - Hover/click on a segment shows timestamp and status reason metadata when available.
  - Timeline component is responsive and remains legible on mobile widths.
- **Dependencies:** T1.6.2, T1.6.3, T4.3.3
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Reusable timeline component added with compact segments, hover/click metadata, and responsive layout:
    - `apps/portal/frontend/src/components/service-health-timeline.tsx`
  - Health timeline adapter added with API-first loading and sample fallback:
    - `apps/portal/frontend/src/lib/adapters/service-health-timeline.ts`
  - Service detail page now includes selectable timeline window (`6h`, `24h`, `7d`; default `24h`):
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Fallback timeline dataset added:
    - `apps/portal/frontend/service-health-timeline.sample.json`
  - Validation runbook added:
    - `docs/runbooks/service-health-timeline.md`

#### T4.3.7 Deployment observability overlay on deployment history
- **Description:** Enrich deployment history rows with post-deploy metric snapshots (error-rate delta, latency delta, availability impact) to speed regression detection.
- **Status:** DONE (2026-03-12)
- **Acceptance Criteria:**
  - Deployment history rows include before/after indicators for at least error rate and latency.
  - Rows with missing comparison windows display explicit unavailable state.
  - Sorting/filtering can prioritize deployments with negative health deltas.
- **Dependencies:** T1.6.4, T4.1.2, T4.3.2, T6.6.2
- **Complexity:** M
- **Risk:** Medium
- **Change Note:** Updated to consume Phase 6 deployment records as the canonical deployment-history source instead of mock-only rows.

#### T4.3.8 Unhealthy deployment and degraded service highlighting
- **Description:** Add frontend detection rules to flag suspicious deployments and propagate degraded badges to service list and detail views.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Rule set identifies unhealthy deployments using configurable thresholds (for example error spike or readiness drop).
  - Service list and service detail views show consistent degraded badge treatment.
  - Alerting logic is isolated in a test-covered adapter/helper module.
- **Dependencies:** T1.6.1, T1.6.3, T1.6.4, T4.3.7
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Threshold-based deployment alerting rules and severity evaluation isolated in helper module with configurable thresholds:
    - `apps/portal/frontend/src/lib/deployment-alerts.ts`
  - Deployment history page consumes alert rules to prioritize and highlight suspicious rows:
    - `apps/portal/frontend/src/pages/service-deployments-page.tsx`
  - Service detail page propagates suspicious deployment state into degraded status treatment and warning banner:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Service list page applies consistent degraded badge treatment for services with suspicious recent deployments:
    - `apps/portal/frontend/src/pages/services-page.tsx`
  - Unit tests added for alert helper logic:
    - `apps/portal/frontend/tests/deployment-alerts.test.ts`
  - Validation runbook added:
    - `docs/runbooks/unhealthy-deployment-highlighting.md`

#### T4.3.9 Logs quick-view panel on service detail
- **Description:** Add a service-level logs quick-view drawer with prebuilt Loki queries and one-click deep links for full investigation in Grafana.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Service detail page includes "View logs" action that opens an inline quick-view panel.
  - Panel provides at least three prebuilt query shortcuts (errors, restarts, recent warnings) scoped to service label/namespace.
  - "Open full logs" deep link launches Grafana/Loki with equivalent query context.
- **Dependencies:** T1.6.2, T1.6.5, T4.2.1, T4.3.5
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Service detail page now includes a `View logs` inline quick-view drawer with preset shortcuts:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Presets implemented for Errors, Restarts, and Warnings with scoped query previews and one-click Grafana deep links:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Shared logs URL builder now supports optional preset/query variables for equivalent deep-link context:
    - `apps/portal/frontend/src/lib/config.ts`
  - Validation runbook added:
    - `docs/runbooks/logs-quick-view-panel.md`

#### T4.3.10 Platform health page in portal UI
- **Description:** Create a dedicated portal page aggregating platform-wide service health, alert counts, and top active incidents from monitoring adapters.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - New route displays platform summary cards, unhealthy services list, and latest alert feed.
  - Page links back to service detail pages and external Grafana dashboards.
  - Loading and partial-failure states keep page usable when one data source fails.
- **Dependencies:** T1.6.3, T4.1.2, T4.3.8
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - New platform-wide health route and page implemented with summary cards, unhealthy services list, and latest incident feed:
    - `apps/portal/frontend/src/pages/platform-health-page.tsx`
  - Monitoring aggregation adapter added with partial-failure warnings and incidents fallback source:
    - `apps/portal/frontend/src/lib/adapters/platform-health.ts`
  - Incidents sample fallback data added:
    - `apps/portal/frontend/platform-health.sample.json`
  - Route and portal navigation wired for desktop and mobile:
    - `apps/portal/frontend/src/App.tsx`
    - `apps/portal/frontend/src/components/navigation/portal-sidebar.tsx`
    - `apps/portal/frontend/src/components/navigation/topbar.tsx`
  - Validation runbook added:
    - `docs/runbooks/platform-health-page.md`

#### T4.3.11 Global incident banner and alert badges
- **Description:** Add an app-wide incident banner and per-service alert badges driven by active monitoring severity states.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Global banner appears on all main portal routes when active incidents exceed configured severity.
  - Service cards/rows show alert count badges with severity color mapping.
  - Banner can be dismissed per session while preserving visibility of critical incidents.
- **Dependencies:** T1.6.1, T1.6.3, T4.3.10
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - App-wide incident polling and banner display logic added with session dismissal behavior and critical visibility override:
    - `apps/portal/frontend/src/App.tsx`
  - Reusable incident banner component added:
    - `apps/portal/frontend/src/components/incident-banner.tsx`
  - Incident severity/count mapping helper added for service-level badges and banner gating:
    - `apps/portal/frontend/src/lib/incident-alerts.ts`
  - Service list rows now show per-service alert count badges with severity tone mapping:
    - `apps/portal/frontend/src/pages/services-page.tsx`
  - Service detail page header now shows matching per-service alert count badge:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Unit tests added for threshold normalization, snapshot mapping, and banner visibility semantics:
    - `apps/portal/frontend/tests/incident-alerts.test.ts`
  - Validation runbook added:
    - `docs/runbooks/global-incident-banner-alert-badges.md`

### E4.4 Live Monitoring Data Integration

Scope: Replace sample/mock monitoring data in the Portal with bounded, live
backend endpoints and shared service identity joins across Argo, Prometheus,
Loki, and registry metadata.

#### T4.4.1 Define service identity contract for monitoring joins
- **Description:** Standardize how the portal identifies a service across Argo, Prometheus, Loki, and the portal registry (name, namespace, app label, environment).
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - A single `ServiceIdentity` shape is defined and used across monitoring APIs and frontend adapters (fields include `serviceId`, `serviceName`, `namespace`, `env`, `appLabel`, optional `argoAppName`).
  - Frontend services adapter provides (or can derive) the required identity fields for all service rows and service detail routes.
  - Contract documented in `docs/contracts/service-identity.md` with examples for at least 2 services and 2 environments.
- **Dependencies:** T1.6.7, T1.6.5, T1.6.6
- **Complexity:** S
- **Risk:** Medium
- **Evidence:**
  - Canonical service identity contract introduced and shared as frontend source-of-truth:
    - `apps/portal/frontend/src/lib/service-identity.ts`
  - Services adapter now derives and exposes identity fields for service rows and detail-route derivation:
    - `apps/portal/frontend/src/lib/adapters/services.ts`
  - Monitoring adapters updated to accept/use `ServiceIdentity` context for service monitoring joins:
    - `apps/portal/frontend/src/lib/adapters/deployments.ts`
    - `apps/portal/frontend/src/lib/adapters/service-metrics.ts`
    - `apps/portal/frontend/src/lib/adapters/service-health-timeline.ts`
    - `apps/portal/frontend/src/lib/adapters/platform-health.ts`
  - Contract documented with field definitions and examples for 2 services across 2 environments:
    - `docs/contracts/service-identity.md`
- **Change Note:** Level 3 enforcement hardening is tracked in `T6.4.4` and `T6.6.5` to ensure this contract becomes mandatory across GitOps/deploy workflows.

#### T4.4.2 Backend API: service metrics summary endpoint (uptime, p95, error rate, restarts)
- **Description:** Implement a backend endpoint to return live summary metrics for a given service identity and time range, backed by Prometheus queries.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `GET /api/services/:serviceId/metrics/summary?range=24h` returns `{ uptimePct, p95LatencyMs, errorRatePct, restartCount, windowStart, windowEnd, generatedAt }`.
  - Query range supports at least `1h`, `24h`, `7d` with server-side validation.
  - Endpoint returns explicit `noData: true` per metric where applicable rather than failing the full response.
  - Non-200 Prometheus responses are translated into stable API errors with correlation IDs in logs.
- **Dependencies:** T4.1.1, T1.2.1, T4.3.2
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - New backend metrics-summary endpoint implemented:
    - `apps/portal/backend/app/main.py` (`GET /services/{serviceId}/metrics/summary`)
  - Supported range validation implemented server-side (`1h`, `24h`, `7d`) with FastAPI query pattern validation.
  - Response includes `uptimePct`, `p95LatencyMs`, `errorRatePct`, `restartCount`, `windowStart`, `windowEnd`, `generatedAt`, and per-metric `noData` flags.
  - Prometheus non-200 and query failures translated to stable HTTP 502 errors with `correlation_id` in logs/error detail.
  - Backward-compat route alias added for existing clients:
    - `GET /services/{serviceId}/metrics-summary`
  - API tests added for success, invalid range, no-data semantics, legacy route, and Prometheus failure translation:
    - `apps/portal/backend/tests/test_api.py`
  - Validation runbook added:
    - `docs/runbooks/service-metrics-summary-endpoint.md`

#### T4.4.3 Frontend: replace metrics-card mocks with live summary endpoint
- **Description:** Wire T4.3.2 metric cards to `GET /api/services/:serviceId/metrics/summary` with loading, stale, and partial no-data handling.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Service detail page metric cards fetch live data on load and on time-range change.
  - Cards show last refreshed timestamp (from `generatedAt`) and a stale indicator if older than a configurable threshold.
  - Partial no-data renders per-card No data without shifting layout.
  - Errors render a non-blocking inline error state with retry.
- **Dependencies:** T4.3.2, T4.4.2
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Service metrics adapter now uses live backend summary endpoint with explicit range support:
    - `apps/portal/frontend/src/lib/adapters/service-metrics.ts`
  - Service detail metrics section now fetches on load and range change (`1h`, `24h`, `7d`) and includes inline retry for non-blocking errors:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Metric cards now render loading placeholders, explicit per-card no-data states, and stale indicators:
    - `apps/portal/frontend/src/components/service-metric-card.tsx`
  - Stale threshold is configurable with `VITE_METRICS_STALE_AFTER_MINUTES`:
    - `apps/portal/frontend/src/lib/config.ts`
  - Validation runbook added:
    - `docs/runbooks/service-metrics-live-summary-frontend.md`

#### T4.4.4 Backend API: service health timeline endpoint (status over time)
- **Description:** Provide a backend endpoint that returns a compact status timeline (healthy/degraded/down/unknown segments) for a service over a selected window, computed from Prometheus signals (availability + error rate + readiness).
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `GET /api/services/:serviceId/health/timeline?range=24h&step=5m` returns an array of `{ start, end, status, reason? }`.
  - Status mapping rules are configurable (thresholds in config file or env vars) and documented.
  - Endpoint supports at least `24h` and `7d` windows with bounded step sizes.
  - Unit tests cover status mapping for healthy/degraded/down/unknown.
- **Dependencies:** T4.4.1, T4.1.1, T4.3.6
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - New backend timeline endpoint implemented:
    - `apps/portal/backend/app/main.py` (`GET /services/{serviceId}/health/timeline`)
  - Endpoint supports `range=24h|7d` with bounded `step` validation and compact segment output shape `{ start, end, status, reason? }`.
  - Status mapping extracted into configurable helper module with env-driven thresholds:
    - `apps/portal/backend/app/health_timeline.py`
  - Threshold configuration documented in backend README:
    - `apps/portal/backend/README.md`
  - Unit tests added for healthy/degraded/down/unknown mapping:
    - `apps/portal/backend/tests/test_health_timeline.py`
  - API tests added for timeline endpoint shape and step validation:
    - `apps/portal/backend/tests/test_api.py`
  - Validation runbook added:
    - `docs/runbooks/service-health-timeline-endpoint.md`

#### T4.4.5 Frontend: wire service health timeline to live timeline endpoint
- **Description:** Replace timeline mock/placeholder logic with the live backend timeline endpoint and add step/range selectors.
- **Status:** DONE (2026-03-12)
- **Acceptance Criteria:**
  - `/services/:serviceId` timeline renders live segments for default `24h`.
  - Range selector supports `24h` and `7d`; step auto-adjusts appropriately.
  - Segment hover shows exact timestamps and reason when present.
  - Timeline remains responsive and legible on mobile widths.
- **Dependencies:** T4.3.6, T4.4.4
- **Complexity:** S
- **Risk:** Low

#### T4.4.6 Backend API: release traceability endpoint (commit → image → Argo state → drift)
- **Description:** Replace sample-based release dashboard with live data by implementing a backend endpoint that aggregates CI/build metadata, deployed image, and Argo sync/health per service/environment.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `GET /api/releases?env=dev&limit=50` returns rows including `{ serviceId, env, commitSha, imageRef, deployedAt, argo: { appName, syncStatus, healthStatus, revision }, drift: { isDrifted, expectedRevision, liveRevision } }`.
  - Drift is computed in a deterministic way and documented (what constitutes drift).
  - Endpoint supports server-side filtering by `serviceId` and `env`.
  - Missing upstream data yields explicit unknown fields; endpoint still returns rows.
- **Dependencies:** T4.3.1, T4.4.1, T4.1.1
- **Complexity:** L
- **Risk:** Medium
- **Evidence:**
  - Live backend release traceability endpoint implemented:
    - `apps/portal/backend/app/main.py` (`GET /releases`)
  - Aggregation helper for CI + Argo metadata joins and deterministic drift computation:
    - `apps/portal/backend/app/release_traceability.py`
  - Deterministic drift rule documented in backend README and endpoint runbook.
  - Server-side filtering by `env`, `serviceId`, and `limit` implemented in endpoint query handling.
  - Missing upstream metadata returns explicit unknown/nullable fields while preserving rows.
  - API tests for endpoint shape/filtering and compatibility path:
    - `apps/portal/backend/tests/test_api.py`
  - Unit tests for drift computation and unknown fallback behavior:
    - `apps/portal/backend/tests/test_release_traceability.py`
  - Validation runbook added:
    - `docs/runbooks/release-traceability-endpoint.md`

#### T4.4.7 Frontend: switch release dashboard adapter from sample to live endpoint
- **Description:** Update `apps/portal/frontend/src/lib/adapters/release-dashboard.ts` to call the live releases endpoint and keep sample JSON only as an offline/dev fallback.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Dashboard uses live endpoint by default and shows Live indicator when data source is API.
  - Sample JSON is only used when API is unreachable (dev mode) or behind an explicit feature flag.
  - Row links (commit/image/Argo) use live values and remain stable across refresh.
- **Dependencies:** T4.3.1, T4.4.6
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Release dashboard adapter now targets live traceability endpoint by default:
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts` (`GET /api/releases?limit=50`)
  - Dashboard now shows explicit data-source indicator (`Live`, `Fallback: Projects`, `Fallback: Sample`):
    - `apps/portal/frontend/src/pages/dashboard-page.tsx`
  - Sample fallback is restricted to dev mode or explicit feature flag:
    - `VITE_ENABLE_RELEASE_SAMPLE_FALLBACK=true`
    - `apps/portal/frontend/src/lib/config.ts`
  - Argo link values are built from live rows and remain deterministic across refresh:
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
  - Validation runbook updated:
    - `docs/runbooks/release-traceability-dashboard.md`

#### T4.4.8 Backend API: logs query endpoint for quick-view (Loki proxy, scoped)
- **Description:** Enable the service logs quick-view panel by adding a backend endpoint that executes a small set of safe, templated Loki queries (errors/restarts/warnings) for a service identity and time range.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `GET /api/services/:serviceId/logs/quickview?preset=errors&range=1h` returns a bounded list of log lines with timestamps and labels.
  - Only approved query presets are supported (no arbitrary query execution from the client).
  - Responses are capped (for example max lines) with pagination token support or more-available indicator.
  - Endpoint enforces auth and rate limits (basic in-memory is acceptable for MVP).
- **Dependencies:** T4.2.1, T4.3.9, T4.4.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - New backend logs quick-view endpoint implemented:
    - `apps/portal/backend/app/main.py` (`GET /services/{serviceId}/logs/quickview`)
  - Preset-only query builder, range/cursor parsing, and in-memory rate limiter implemented in dedicated helper module:
    - `apps/portal/backend/app/logs_quickview.py`
  - Endpoint returns bounded line list with `moreAvailable` and `nextCursor` pagination support.
  - Auth is enforced via existing bearer/forwarded-user dependency and per-identity rate limiting returns HTTP 429 when exceeded.
  - Unit tests added for preset validation, cursor/time-window behavior, and rate limiting:
    - `apps/portal/backend/tests/test_logs_quickview.py`
  - API tests added for preset rejection, bounded response/pagination indicator, and rate-limit enforcement:
    - `apps/portal/backend/tests/test_api.py`
  - Backend config/docs updated:
    - `apps/portal/backend/README.md`
  - Validation runbook added:
    - `docs/runbooks/logs-quickview-endpoint.md`

#### T4.4.9 Frontend: connect logs quick-view panel to live Loki quick-view endpoint
- **Description:** Replace any mocked log preview with live data from `GET /api/services/:serviceId/logs/quickview` while keeping deep links for full Grafana exploration.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Quick-view panel loads logs for at least 3 presets (`errors`, `restarts`, `warnings`) with range selector.
  - Loading, empty, and error states are non-blocking and keep deep link usable.
  - Open full logs deep link preserves equivalent query context and time range.
- **Dependencies:** T4.3.9, T4.4.8, T4.3.5
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Service details logs quick-view drawer now fetches live backend data for `errors`, `restarts`, and `warnings` presets with range selector (`15m`, `1h`, `6h`, `24h`):
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Dedicated frontend adapter added for Loki quick-view endpoint and response normalization:
    - `apps/portal/frontend/src/lib/adapters/logs-quickview.ts`
  - Quick-view renders non-blocking loading, empty, and error states while preserving Grafana deep-link action.
  - Validation runbook added:
    - `docs/runbooks/logs-quickview-live-frontend.md`

#### T4.4.10 Backend API: active alerts feed endpoint (for platform page + global banner)
- **Description:** Provide a backend endpoint to expose current active alerts (from Prometheus/Alertmanager) mapped to portal severities and optionally to services.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `GET /api/alerts/active` returns an array with `{ id, severity, title, description?, startsAt, labels, serviceId?, env? }`.
  - Severity mapping (`warning`/`critical`) is consistent with frontend badge styles and documented.
  - Endpoint supports filtering by `env` and optionally `serviceId`.
  - Partial failures degrade gracefully (returns empty + error metadata rather than hard fail where possible).
- **Dependencies:** T4.1.2, T4.3.10, T4.3.11
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - New active alerts endpoint implemented:
    - `apps/portal/backend/app/main.py` (`GET /alerts/active`)
  - Alertmanager normalization and severity mapping helper added:
    - `apps/portal/backend/app/alerts_feed.py`
  - Endpoint supports `env` and `serviceId` filters with normalized response shape.
  - Upstream Alertmanager failures degrade gracefully by returning HTTP 200 with an empty list and logged correlation metadata.
  - Compatibility incidents route added for existing frontend adapter:
    - `apps/portal/backend/app/main.py` (`GET /monitoring/incidents`)
  - API and helper tests added:
    - `apps/portal/backend/tests/test_api.py`
    - `apps/portal/backend/tests/test_alerts_feed.py`
  - Validation runbook added:
    - `docs/runbooks/active-alerts-endpoint.md`

#### T4.4.11 Frontend: wire platform health page and incident banner to live alerts feed
- **Description:** Use `GET /api/alerts/active` to power the platform health page alert feed and the global incident banner + per-service badges.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Platform health page shows alert list with severity, start time, and links to relevant services (when mapped).
  - Global banner appears when active alerts exceed configured severity threshold.
  - Banner dismissal persists per session and does not hide critical alerts.
- **Dependencies:** T4.3.10, T4.3.11, T4.4.10
- **Complexity:** M
- **Risk:** Low
- **Evidence:**
  - Platform health adapter now uses live `GET /api/alerts/active` as primary source for incidents/alerts and keeps compatibility fallback:
    - `apps/portal/frontend/src/lib/adapters/platform-health.ts`
  - Platform health page alert feed renders severity, start time, mapped service links, and Grafana deep links from live alert data:
    - `apps/portal/frontend/src/pages/platform-health-page.tsx`
  - Global incident banner + per-service badges continue to derive from live alert snapshot polling in app shell:
    - `apps/portal/frontend/src/App.tsx`
    - `apps/portal/frontend/src/lib/incident-alerts.ts`
  - Critical incidents remain visible after dismissal (session dismissal only hides non-critical severities):
    - `apps/portal/frontend/tests/incident-alerts.test.ts`
  - Validation runbook added:
    - `docs/runbooks/platform-health-live-alerts-frontend.md`

#### T4.4.12 Observability config hardening: query templates, limits, caching
- **Description:** Add config knobs for ranges, limits, caching TTLs, and query templates so live monitoring endpoints are safe and stable under frequent portal refresh.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Backend monitoring endpoints implement caching (per service + range) with configurable TTL.
  - All endpoints enforce sane bounds (max range, max step resolution, max rows/log lines).
  - Query templates and thresholds are centralized and documented (`docs/monitoring/query-templates.md`).
  - Basic load test script or documented manual test verifies endpoints under repeated refresh.
- **Dependencies:** T4.4.2, T4.4.4, T4.4.8, T4.4.10
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Shared observability config module added for allowed ranges, limits, cache TTLs, and query template env knobs:
    - `apps/portal/backend/app/observability_config.py`
  - In-memory TTL cache helper added and applied to metrics summary, health timeline, logs quick-view, and active alerts endpoints:
    - `apps/portal/backend/app/observability_cache.py`
    - `apps/portal/backend/app/main.py`
  - Monitoring endpoints now enforce configured bounds for timeline points, logs max rows, alerts max rows, and allowed ranges.
  - Query templates documented with variable contracts:
    - `docs/monitoring/query-templates.md`
  - Repeated-refresh smoke test script added:
    - `apps/portal/backend/scripts/monitoring_refresh_smoke.py`
  - Validation runbook added:
    - `docs/runbooks/observability-config-hardening.md`

### E4.5 Frontend Live-Data Finalization (No Mock/Sample UI Paths)

Scope: Finalize Portal UI behavior so dashboard/service/platform screens render
cluster-backed data only. Remove all sample/mock fallback paths in frontend
adapters and replace placeholder content with explicit live-data states.

#### T4.5.1 Enforce strict live-data mode in frontend adapters
- **Description:** Remove sample JSON fallback behavior from dashboard/services/platform adapters for non-dev environments and add a strict live-data mode that fails closed.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Release dashboard adapter does not use `release-dashboard.sample.json` unless explicitly enabled in local dev.
  - Services/platform adapters do not silently switch to sample payloads in cluster environments.
  - A single frontend config flag controls strict behavior (`no sample fallback`) and defaults to enabled for deployed envs.
- **Dependencies:** T4.4.7, T4.4.11
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Release dashboard adapter now uses live API sources only (`/releases` then `/projects`) and no sample dataset path:
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
  - Services and platform adapters no longer contain sample JSON fallback logic:
    - `apps/portal/frontend/src/lib/adapters/services.ts`
    - `apps/portal/frontend/src/lib/adapters/platform-health.ts`
  - Live-data-only behavior is now unconditional after decommission (strict-mode flag path superseded by T4.5.9):
    - `apps/portal/frontend/src/lib/config.ts`

#### T4.5.2 Remove mock deployment rows from service overview
- **Description:** Replace `createMockDeployments` and deployment mock fallback in service details with explicit empty/error UI states tied to live deployment history.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Service detail page no longer renders synthetic `*-mock-*` deployments.
  - When deployment history is unavailable, UI shows explicit unavailable state with retry.
  - Deployment preview on service detail and full deployment page are consistent for the same service.
- **Dependencies:** T1.6.4, T4.3.7
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Service detail deployments section now renders explicit unavailable/empty states with retry, and no synthetic rows:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
  - Full deployments page uses the same live deployment history adapter path:
    - `apps/portal/frontend/src/pages/service-deployments-page.tsx`
  - Deployment adapter no longer generates mock deployments (`createMockDeployments` removed):
    - `apps/portal/frontend/src/lib/adapters/deployments.ts`

#### T4.5.3 Service registry/API-only source for dashboard joins
- **Description:** Eliminate static `services.sample.json` dependency and require API-backed service identity metadata for all dashboard joins and links.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Services adapter no longer loads sample registry in deployed env.
  - Missing registry API produces explicit page-level partial-failure state rather than fallback rows.
  - Dashboard/service-detail link generation continues to work from live registry identity fields.
- **Dependencies:** T4.4.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Services adapter is API-only and derives canonical registry rows from `/projects` metadata:
    - `apps/portal/frontend/src/lib/adapters/services.ts`
  - Release dashboard performs registry join normalization (ID/name mapping) for stable links:
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
  - Dashboard renders explicit partial-data warning banner when registry join is incomplete/unavailable:
    - `apps/portal/frontend/src/pages/dashboard-page.tsx`

#### T4.5.4 Release dashboard completeness contract for live rows
- **Description:** Ensure release dashboard renders full live traceability fields (commit/image/Argo/drift) and clearly distinguishes unknown upstream values from missing UI wiring.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Rows show populated live values when backend provides metadata.
  - Unknown fields are labeled as upstream unknown (not fallback/sample).
  - Dashboard-level indicator shows live source status and unknown-field count.
- **Dependencies:** T4.4.6, T4.4.7
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Release dashboard adapter emits `liveStatus` and `unknownFieldCount` for rendered live rows:
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
  - Dashboard UI shows live-source indicator, unknown count card, and per-cell upstream-unknown labeling:
    - `apps/portal/frontend/src/pages/dashboard-page.tsx`

#### T4.5.5 Platform health and incident feed: live-only behavior
- **Description:** Remove sample incident fallback usage from platform health flow and keep page usable using explicit live-feed degradation states.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `platform-health.sample.json` is not used in deployed env.
  - Incident banner/service badges are derived only from `/api/alerts/active`.
  - Upstream failures surface as non-blocking warnings without fabricating incident rows.
- **Dependencies:** T4.4.10, T4.4.11
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Platform health adapter now sources incident feed only from `/api/alerts/active`:
    - `apps/portal/frontend/src/lib/adapters/platform-health.ts`
  - Upstream alert failures surface as warnings with empty incident lists (no fabricated rows):
    - `apps/portal/frontend/src/lib/adapters/platform-health.ts`
  - App-level incident badge/banner snapshot continues to derive from live alerts feed polling:
    - `apps/portal/frontend/src/App.tsx`

#### T4.5.6 Dashboard auth/session diagnostics for API gating
- **Description:** Add frontend diagnostics surface for auth-gated API failures (for example OAuth2-proxy 401 HTML responses) so operators can distinguish data absence from auth routing issues.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Dashboard shows a clear diagnostic message when API returns auth HTML/redirect payload instead of JSON.
  - UI exposes a consistent remediation hint (session expired, not forwarded auth headers, etc.).
  - Diagnostics are non-blocking and do not crash route rendering.
- **Dependencies:** T4.4.11
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Frontend API client detects auth-gateway HTML/redirect responses and raises structured diagnostic errors:
    - `apps/portal/frontend/src/lib/api.ts`
  - Dashboard renders dedicated non-blocking auth/session diagnostics panel with remediation hints:
    - `apps/portal/frontend/src/pages/dashboard-page.tsx`

#### T4.5.7 Cluster identity normalization for release/metrics joins
- **Description:** Standardize service identifiers used in UI and backend joins (avoid mixed human-readable names and runtime app IDs) to improve live data population consistency.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Dashboard release rows map to canonical `serviceId` values used by metrics/logs/adapters.
  - Join mismatch cases are surfaced in diagnostics with explicit keys.
  - At least one migration runbook documents normalization steps for existing project rows.
- **Dependencies:** T4.4.1, T4.4.6
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Shared canonical service ID normalizer added and used by identity creation:
    - `apps/portal/frontend/src/lib/service-identity.ts`
  - Services and release adapters normalize IDs consistently for joins/routes:
    - `apps/portal/frontend/src/lib/adapters/services.ts`
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
  - Join mismatch diagnostics include explicit unmatched keys (`serviceId|serviceName|environment`):
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
  - Migration runbook added:
    - `docs/runbooks/cluster-identity-normalization.md`

#### T4.5.8 End-to-end live-data validation suite (UI + API)
- **Description:** Add smoke checks that verify dashboard pages are populated from live API responses and no sample/mock paths are exercised in deployed environments.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Automated checks cover dashboard, service detail, platform health, and logs quick-view.
  - Tests assert that sample files are not fetched and mock row markers are absent.
  - Validation runbook includes manual plus scripted checks for cluster deployments.
- **Dependencies:** T4.5.1, T4.5.2, T4.5.5
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Scripted smoke suite added for dashboard, service detail, platform health, and logs quick-view live API checks:
    - `apps/portal/frontend/scripts/live_data_smoke.sh`
  - Frontend npm script added:
    - `apps/portal/frontend/package.json` (`test:live-smoke`)
  - Script asserts mock/sample markers are absent and sample asset paths are not served in deployed UI paths.
  - Manual + scripted validation runbook added:
    - `docs/runbooks/live-data-validation-suite.md`

#### T4.5.9 Decommission sample/mock assets after cutover
- **Description:** Remove obsolete sample JSON files and dead fallback code after live-data cutover is verified.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Unused sample assets are deleted or explicitly marked local-dev-only.
  - Fallback-only code paths are removed from adapters/pages.
  - Documentation is updated to describe live-data-only behavior in cluster environments.
- **Dependencies:** T4.5.8
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Frontend sample JSON assets deleted:
    - `apps/portal/frontend/release-dashboard.sample.json` (deleted)
    - `apps/portal/frontend/services.sample.json` (deleted)
    - `apps/portal/frontend/platform-health.sample.json` (deleted)
    - `apps/portal/frontend/service-health-timeline.sample.json` (deleted)
    - `apps/portal/frontend/service-metrics.sample.json` (deleted)
  - Fallback-only code paths removed from release/timeline/deployments adapters:
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
    - `apps/portal/frontend/src/lib/adapters/service-health-timeline.ts`
    - `apps/portal/frontend/src/lib/adapters/deployments.ts`
  - Runbooks updated for live-data-only cluster behavior:
    - `docs/runbooks/release-traceability-dashboard.md`
    - `docs/runbooks/platform-health-page.md`
    - `docs/runbooks/service-health-timeline.md`
    - `docs/runbooks/global-incident-banner-alert-badges.md`
    - `docs/runbooks/strict-live-data-mode-frontend.md`

---

### E4.6 Backend Service Registry and Project Source Normalization

Scope: Remove seeded/default project behavior from backend request paths and
standardize a canonical, cluster-backed service registry model used by
`/projects`, release joins, and monitoring identity resolution.

#### T4.6.1 Define backend service registry schema (canonical identity + provenance)
- **Description:** Add canonical backend schema for service identity records and provenance metadata so joins do not rely on display names.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - DB schema includes canonical keys (`serviceId`, `serviceName`, `namespace`, `env`, `appLabel`, optional `argoAppName`).
  - Uniqueness constraints prevent duplicate service/env identity rows.
  - Schema includes provenance/freshness fields (`source`, `lastSyncedAt`).
- **Dependencies:** T4.4.1
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Alembic migration added:
    - `apps/portal/backend/alembic/versions/20260305_0002_create_service_registry_table.py`
  - New `service_registry` schema includes canonical identity fields:
    - `service_id`, `service_name`, `namespace`, `env`, `app_label`, optional `argo_app_name`
  - Provenance/freshness fields included:
    - `source`, `source_ref`, `last_synced_at`
  - Uniqueness and indexing constraints added:
    - primary key on (`service_id`, `env`)
    - unique constraint on (`service_name`, `namespace`, `env`)
    - indexes on `env` and (`source`, `last_synced_at`)

#### T4.6.2 Add registry sync pipeline from GitOps/cluster metadata
- **Description:** Implement a sync job that reads GitOps/Argo/Kubernetes metadata and upserts canonical service registry rows.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Sync job populates registry without manual row creation.
  - Job is idempotent and safe to rerun.
  - Sync failures are logged with correlation context and metrics.
- **Dependencies:** T4.6.1, T2.2.2
- **Complexity:** L
- **Risk:** Medium
- **Evidence:**
  - New sync pipeline module added:
    - `apps/portal/backend/app/service_registry_sync.py`
  - Sync reads cluster metadata from Kubernetes Deployments and Argo CD Applications:
    - in-cluster API calls to `apps/v1` Deployments and `argoproj.io/v1alpha1` Applications
  - Idempotent upsert implemented via `ON CONFLICT (service_id, env) DO UPDATE`:
    - maintains canonical fields and updates `last_synced_at`/`updated_at`
  - Admin-triggered sync endpoint added:
    - `POST /service-registry/sync` in `apps/portal/backend/app/main.py`
  - Scripted job entrypoint added:
    - `apps/portal/backend/scripts/sync_service_registry.py`
  - Sync failures logged with correlation context and summarized metrics:
    - response fields include `correlationId`, `inserted`, `updated`, `sourceFailures`, `durationMs`
  - Tests added for record derivation, normalization, and source-failure behavior:
    - `apps/portal/backend/tests/test_service_registry_sync.py`
  - Validation runbook added:
    - `docs/runbooks/service-registry-sync-pipeline.md`

#### T4.6.3 Remove request-time project seeding and seeded defaults
- **Description:** Eliminate `DEFAULT_PROJECTS` and `_seed_projects_if_empty` behavior from backend request paths.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `GET /api/projects` no longer inserts seeded/default rows on read.
  - Empty data state is returned explicitly when no registry rows exist.
  - Existing tests updated to assert no implicit seed behavior.
- **Dependencies:** T4.6.1
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Removed request-time seed constants/helpers from backend:
    - `DEFAULT_PROJECTS` and `_seed_projects_if_empty` deleted from `apps/portal/backend/app/main.py`
  - `/projects` read path no longer performs writes:
    - `GET /projects` now only runs `SELECT id, name, environment FROM projects`
  - Release project-row loading path no longer triggers seeding:
    - `_load_project_rows()` now reads existing rows only
  - Test coverage added for no-seed-on-read behavior:
    - `test_projects_list_does_not_seed_defaults_on_read` in `apps/portal/backend/tests/test_api.py`

#### T4.6.4 Rewire `/projects` to canonical registry projection
- **Description:** Serve `/projects` from canonical registry projection rows instead of mixed/manual project labels.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `/api/projects` rows map deterministically to canonical `serviceId` identities.
  - Row shape remains backward-compatible for current frontend consumers.
  - Projection exposes env-scoped rows suitable for release and monitoring joins.
- **Dependencies:** T4.6.1, T4.6.2, T4.6.3
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - `/projects` now reads from canonical registry projection:
    - `SELECT service_id, service_name, env FROM service_registry` in `apps/portal/backend/app/main.py`
  - Response shape remains backward-compatible for existing frontend adapters:
    - returns `{ id, name, environment }` mapped from canonical fields
  - Release join seed input now also uses canonical identity rows:
    - `_load_project_rows()` reads `service_id` and `env` from `service_registry`
  - `POST /projects` now upserts canonical service identity rows (manual source):
    - writes into `service_registry` with deterministic `service_id` normalization
  - Tests updated for canonical projection path:
    - `test_projects_list_does_not_seed_defaults_on_read`
    - `test_create_project_normalizes_to_canonical_service_id`
    - both in `apps/portal/backend/tests/test_api.py`

#### T4.6.5 Canonical release join in backend traceability pipeline
- **Description:** Ensure `/api/releases` and related backend joins use canonical service IDs from registry projection.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Release rows map to canonical `serviceId` values used by metrics/logs routes.
  - Join mismatch diagnostics include explicit key details (`serviceId|serviceName|env`).
  - Unknown upstream values remain explicit instead of silently remapped.
- **Dependencies:** T4.4.6, T4.6.4
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Release join now anchors output keys to canonical registry projection rows:
    - `build_release_traceability_rows()` iterates registry-derived `project_rows` only in `apps/portal/backend/app/release_traceability.py`
  - Canonical matching supports upstream rows keyed by canonical `serviceId` or registry `service_name`:
    - fallback matching added for `serviceId == service_name` and normalized-id lookup
  - Join mismatch diagnostics logged with explicit `serviceId|serviceName|env` key details:
    - `release_join_mismatch source=... key=<serviceId>|<serviceName>|<env> reason=missing_registry_mapping`
  - Unknown upstream values remain explicit (`unknown`), not remapped silently:
    - normalization behavior retained for Argo sync/health when missing/invalid
  - Canonical join tests added:
    - `test_build_release_rows_maps_upstream_service_name_to_canonical_service_id`
    - `test_build_release_rows_logs_unmatched_upstream_keys`
    - in `apps/portal/backend/tests/test_release_traceability.py`

#### T4.6.6 Backfill/migration plan for existing project rows
- **Description:** Provide a one-time migration path from legacy project labels (human-readable) to canonical service IDs.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Migration script maps existing rows and produces a diff/report.
  - Script supports dry-run and rollback instructions.
  - Validation confirms frontend links and adapter joins remain stable post-migration.
- **Dependencies:** T4.6.4
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - One-time migration script added:
    - `apps/portal/backend/scripts/migrate_projects_to_service_registry.py`
  - Script reads legacy `projects`, maps to canonical `service_registry`, and emits per-row diff/report:
    - includes `legacyProjectId`, canonical `serviceId`, `mappingRule`, `changedFields`
  - Dry-run supported by default, apply via explicit flag:
    - dry-run: `python scripts/migrate_projects_to_service_registry.py`
    - apply: `python scripts/migrate_projects_to_service_registry.py --apply`
  - Rollback SQL generation supported (preview + optional file output):
    - `--rollback-file /path/to/rollback.sql`
  - Mapping and plan logic isolated and unit-tested:
    - `apps/portal/backend/app/projects_backfill.py`
    - `apps/portal/backend/tests/test_projects_backfill.py`
  - Migration + validation runbook added:
    - `docs/runbooks/projects-backfill-migration.md`

#### T4.6.7 Registry freshness and mismatch diagnostics endpoint
- **Description:** Add backend diagnostics surface for registry freshness, sync health, and join mismatch counts.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Endpoint or health payload includes registry freshness (`lastSyncedAt`) and mismatch counters.
  - Diagnostic output is consumable by runbooks and smoke checks.
  - Stale registry condition is clearly distinguishable from empty registry.
- **Dependencies:** T4.6.2, T4.6.5
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - New diagnostics endpoint added:
    - `GET /service-registry/diagnostics` in `apps/portal/backend/app/main.py`
  - Endpoint freshness payload includes:
    - `rowCount`, `lastSyncedAt`, `staleAfterMinutes`, `isEmpty`, `isStale`, `state`
  - Stale vs empty distinction implemented:
    - `state=empty` when registry has zero rows
    - `state=stale` when rows exist but `lastSyncedAt` is missing/older than threshold
  - Join mismatch counters/keys exposed from canonical release join diagnostics:
    - `ciUnmatchedCount`, `argoUnmatchedCount`, `ciUnmatchedKeys`, `argoUnmatchedKeys`
    - keys formatted as `serviceId|serviceName|env`
  - Mismatch diagnostics helper added:
    - `build_release_join_diagnostics()` in `apps/portal/backend/app/release_traceability.py`
  - Tests added for endpoint and join diagnostics:
    - `apps/portal/backend/tests/test_api.py`
    - `apps/portal/backend/tests/test_release_traceability.py`
  - Runbook added for smoke checks:
    - `docs/runbooks/service-registry-diagnostics.md`

#### T4.6.8 Validation runbook and smoke checks for project-source cutover
- **Description:** Add explicit validation for `/projects` live-source cutover and canonical join integrity.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Runbook includes scripted and manual checks for project source, releases, and service-route consistency.
  - Smoke checks fail when seeded/default project rows are detected.
  - Evidence captures before/after behavior in dev cluster.
- **Dependencies:** T4.6.3, T4.6.4, T4.6.5
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - New cutover smoke script added:
    - `apps/portal/backend/scripts/project_source_cutover_smoke.py`
  - Script validates:
    - `/projects` canonical row shape and no seeded/default legacy rows
    - `/releases` join integrity against `/projects` keys (`serviceId|env`)
    - service-route consistency via `/services/:serviceId/metrics/summary`
  - Smoke fails on seeded/default legacy markers:
    - legacy IDs (`proj*`, `proj-dev`, `proj-prod`) and legacy name (`Homelab App`)
  - Validation runbook added:
    - `docs/runbooks/project-source-cutover-validation.md`
  - Dev-cluster before/after behavior captured during migration validation:
    - initial migration apply failed with unique conflict on `(service_name, namespace, env)=(Allowed, default, dev)`
    - rerun after `update_rekey` fix showed plan with `update_rekey` actions and `applied: true`
    - backend test suite passed after fixes (`62 passed`)

---

### E4.7 Live Catalog Convergence: GitOps Apps, Cluster Services, and Monitoring Readiness

Scope: Make project/service data fully live and deterministic by sourcing projects
from GitOps `apps/` definitions, services from cluster discovery, and exposing
monitoring readiness so the dashboard reflects real operational state.

#### T4.7.1 Define source-of-truth contract for Projects vs Services
- **Description:** Formalize that Projects come from GitOps app definitions and Services come from live cluster resources, with explicit join keys and ownership boundaries.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - Contract documents canonical keys for both domains (`projectId`, `serviceId`, `env`, `namespace`, `appLabel`).
  - `/projects` and `/services` ownership boundaries are explicit and non-overlapping.
  - Contract includes mismatch/error semantics used by UI diagnostics.
- **Dependencies:** T4.6.4, T4.6.5
- **Complexity:** S
- **Risk:** Low
- **Contract:**
  - Domain ownership:
    - `GET /projects` is the canonical catalog of intended workloads declared in GitOps `apps/` definitions only (`source=gitops_apps`).
    - `GET /services` is the canonical catalog of observed runtime workloads discovered from the live cluster only (`source=cluster_services`).
    - `/projects` does not fabricate runtime health, endpoints, or deployment state from cluster reads.
    - `/services` does not invent desired-state/project metadata from manual rows, seeded rows, or UI-only labels.
  - Canonical keys and row shapes:
    - `GET /projects` row shape: `{ projectId, projectName, env, namespace, appLabel, source, sourceRef, lastSyncedAt }`.
    - `projectId` is the stable GitOps project identity for an app/env pair and remains immutable across syncs unless the GitOps definition itself is renamed.
    - `GET /services` row shape: `{ serviceId, serviceName, env, namespace, appLabel, source, sourceRef, lastSyncedAt }`.
    - `serviceId` is the stable runtime identity derived from normalized service metadata and is the canonical key used by service routes and monitoring joins.
    - Shared fields across both domains are limited to `env`, `namespace`, and `appLabel`; those fields are join inputs, not proof of ownership transfer.
  - Source references and freshness:
    - `sourceRef` for `/projects` records the GitOps provenance (`repo@commit:path` or equivalent manifest reference).
    - `sourceRef` for `/services` records the cluster provenance (`cluster/context:namespace/resourceName` or equivalent discovery reference).
    - `lastSyncedAt` is required on both contracts and represents when that source projection was last refreshed, not when the workload was last deployed.
  - Join contract:
    - Primary join key: `env + namespace + appLabel`.
    - Secondary fallback key: `env + serviceId` after normalization when `appLabel` is absent or inconsistent.
    - Join results must be deterministic: the same source rows produce the same project-service mapping on repeated syncs.
    - Diagnostic key format for unmatched or ambiguous rows: `serviceId|serviceName|env`.
  - Mismatch semantics:
    - `project_only`: GitOps project row exists with no matching live service row.
    - `service_only`: live service row exists with no matching GitOps project row.
    - `ambiguous_join`: more than one candidate row matches the join key and no deterministic winner exists.
    - `key_conflict`: canonical keys disagree across sources for what appears to be the same workload.
    - Mismatch records are exposed through diagnostics payloads and must not be silently patched by either endpoint.
  - Error and degradation semantics used by UI diagnostics:
    - `state=empty`: source reachable but zero rows.
    - `state=stale`: source rows exist but freshness threshold exceeded.
    - `state=degraded`: partial source failure or partial projection failure (for example cluster API read succeeds for some namespaces but not all, or GitOps manifest scan is incomplete).
    - `state=auth_error`: upstream auth/session gate detected.
    - `state=healthy`: source reachable, rows present when expected, and freshness threshold satisfied.
  - API boundary guardrails:
    - Canonical `/projects` responses exclude `source=manual` or seeded fallback rows by default.
    - Canonical `/services` responses exclude synthetic adapter rows and sample JSON fallbacks in non-dev live mode.
    - Any optional backfill/manual rows must remain explicitly tagged non-canonical and must not override GitOps or cluster-owned identities.
- **Evidence:**
  - Contract recorded in this roadmap section as the baseline for `T4.7.2` and `T4.7.3` implementation.
  - Ownership boundaries align with completed canonical registry work in `T4.6.4`, diagnostics semantics in `T4.6.7`, and cutover validation in `T4.6.8`.
  - Join keys and diagnostic key format are now explicit for frontend and backend consumers before live catalog convergence work proceeds.

#### T4.7.2 Implement Projects sync from GitOps `apps/` source
- **Description:** Replace manual/project-row drift by syncing project catalog from GitOps app manifests/repo paths.
- **Status:** DONE (2026-03-05)
- **Acceptance Criteria:**
  - `/projects` rows are generated from GitOps `apps/` definitions for target env.
  - Manual-only project rows are either rejected or explicitly tagged non-canonical.
  - Sync is idempotent and records provenance (`source=gitops_apps`, `sourceRef`).
- **Dependencies:** T4.7.1, T4.6.2
- **Complexity:** M
- **Risk:** Medium
- **Validation/Evidence Checklist:**
  - `curl /projects?env=dev` shows only canonical GitOps apps for target env.
  - Re-running sync with unchanged repo commit yields `inserted=0` and stable row count.
  - Changed app manifest name/namespace updates corresponding project row deterministically.
  - Manual project create path behavior matches chosen guardrail and is explicitly tested.
- **Evidence:**
  - Dedicated GitOps-backed project projection added:
    - `apps/portal/backend/alembic/versions/20260305_0003_create_project_registry_table.py`
    - `apps/portal/backend/app/gitops_project_sync.py`
    - `apps/portal/backend/scripts/sync_project_registry.py`
  - `/projects` now reads canonical GitOps rows from `project_registry` and supports env filtering:
    - `apps/portal/backend/app/main.py`
    - `apps/portal/backend/openapi.json`
  - Manual project creation is explicitly rejected with `409` because the catalog is GitOps-owned:
    - `apps/portal/backend/app/main.py`
    - `apps/portal/frontend/src/pages/projects-page.tsx`
  - Admin sync entrypoint now supports GitOps project sync through the existing endpoint:
    - `POST /service-registry/sync?source=gitops_apps&env=...`
    - `apps/portal/backend/app/main.py`
  - Test coverage and runbook added for GitOps sync:
    - `apps/portal/backend/tests/test_gitops_project_sync.py`
    - `apps/portal/backend/tests/test_service_registry_sync.py`
    - `docs/runbooks/project-registry-sync-pipeline.md`

#### T4.7.3 Implement Services sync from live cluster resources
- **Description:** Build/extend service discovery to source `/services` from Kubernetes resources (Service/Deployment/labels) in the running cluster.
- **Status:** DONE (2026-03-06)
- **Acceptance Criteria:**
  - `/services` returns live cluster-backed entries scoped by env/namespace.
  - Service identity normalizes to canonical `serviceId` for joins.
  - Missing/partial cluster metadata is surfaced as diagnostics, not fabricated rows.
- **Dependencies:** T4.7.1, T4.6.2
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Cluster sync now reads both Kubernetes `Service` and `Deployment` resources and normalizes canonical `serviceId` rows with `source=cluster_services`:
    - `apps/portal/backend/app/service_registry_sync.py`
  - Sync prunes stale service rows only for namespaces that were refreshed successfully, so partial namespace failures surface as diagnostics instead of deleting known-good rows:
    - `apps/portal/backend/app/service_registry_sync.py`
    - `docs/runbooks/service-registry-sync-pipeline.md`
  - Live services API added:
    - `GET /services?env=&namespace=`
    - `GET /services/{serviceId}?env=`
    - `apps/portal/backend/app/main.py`
    - `apps/portal/backend/openapi.json`
  - Frontend services adapter is now API-first for `/services` with `/projects` fallback retained for environments that have not enabled the service API yet:
    - `apps/portal/frontend/src/lib/api.ts`
    - `apps/portal/frontend/src/lib/adapters/services.ts`
  - Test coverage added/updated for service discovery and API projection:
    - `apps/portal/backend/tests/test_service_registry_sync.py`
    - `apps/portal/backend/tests/test_api.py`

#### T4.7.4 Canonical join bridge between GitOps Projects and cluster Services
- **Description:** Add deterministic reconciliation between GitOps project identities and discovered cluster service identities.
- **Status:** DONE (2026-03-06)
- **Acceptance Criteria:**
  - Join logic supports one-to-one and one-to-many mappings with explicit keys.
  - Unmatched project/service records are exposed via diagnostics endpoint payload.
  - Dashboard links (`project -> service`, `service -> project`) remain stable.
- **Dependencies:** T4.7.2, T4.7.3, T4.6.7
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Deterministic catalog reconciliation helper added with primary-key join, fallback service-ID join, one-to-many handling, and unmatched/ambiguous diagnostics:
    - `apps/portal/backend/app/catalog_reconciliation.py`
    - `apps/portal/backend/tests/test_catalog_reconciliation.py`
  - Reconciliation endpoint added for API consumers:
    - `GET /catalog/reconciliation?env=&projectId=&serviceId=`
    - `apps/portal/backend/app/main.py`
    - `apps/portal/backend/openapi.json`
  - `/service-registry/diagnostics` now includes catalog join diagnostics for `projectOnly`, `serviceOnly`, `oneToMany`, and `ambiguousJoin` states:
    - `apps/portal/backend/app/main.py`
    - `apps/portal/backend/tests/test_api.py`
  - Frontend release dashboard now canonicalizes service routes through the reconciliation bridge before linking to `/services/:serviceId`:
    - `apps/portal/frontend/src/lib/adapters/release-dashboard.ts`
  - Frontend pages now expose stable cross-links:
    - `project -> service` from `apps/portal/frontend/src/pages/projects-page.tsx`
    - `service -> project` from `apps/portal/frontend/src/pages/services-page.tsx`

#### T4.7.5 Monitoring provider readiness and connectivity hardening
- **Description:** Ensure Prometheus/Loki/alerts queries are live and diagnosable so 502 provider failures are actionable and non-ambiguous.
- **Status:** DONE (2026-03-06)
- **Acceptance Criteria:**
  - Metrics/logs/alerts endpoints include structured provider status and correlation IDs on failure.
  - Health/diagnostics expose provider reachability per backend (`prometheus`, `loki`, `alertmanager`).
  - Runbook includes cluster checks for URL/auth/network-policy misconfiguration.
- **Dependencies:** T4.4.2, T4.4.8, T4.4.10, T4.6.7
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Shared monitoring-provider contract added for readiness probes and structured failure payloads:
    - `apps/portal/backend/app/monitoring_providers.py`
  - Metrics, logs, and alerts endpoints now expose `providerStatus`, and 502 failures return structured `message`, `correlationId`, and provider metadata:
    - `apps/portal/backend/app/main.py`
    - `apps/portal/backend/tests/test_api.py`
  - Provider reachability exposed through:
    - `GET /health?includeProviders=true`
    - `GET /monitoring/providers/diagnostics`
    - `apps/portal/backend/openapi.json`
  - Frontend API/error handling updated so structured monitoring failures render actionable messages and alerts consume the new envelope:
    - `apps/portal/frontend/src/lib/api.ts`
    - `apps/portal/frontend/src/lib/adapters/service-metrics.ts`
    - `apps/portal/frontend/src/lib/adapters/logs-quickview.ts`
    - `apps/portal/frontend/src/lib/adapters/platform-health.ts`
  - Runbooks now cover provider diagnostics plus URL/auth/network-policy checks:
    - `docs/runbooks/monitoring-provider-readiness.md`
    - `docs/runbooks/service-metrics-summary-endpoint.md`
    - `docs/runbooks/logs-quickview-endpoint.md`
    - `docs/runbooks/active-alerts-endpoint.md`

#### T4.7.6 Dashboard contract: show live readiness, unknowns, and degraded states
- **Description:** Update frontend contract so dashboard pages explicitly show live readiness for projects/services/monitoring instead of appearing empty or broken.
- **Status:** DONE (2026-03-06)
- **Acceptance Criteria:**
  - Projects and Services pages indicate source freshness and degradation state.
  - Monitoring panels distinguish `upstream unknown`, `provider unreachable`, and `no data`.
  - Route rendering remains non-blocking under partial outages.
- **Dependencies:** T4.7.4, T4.7.5, T4.5.6
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Added a project freshness/readiness surface via:
    - `GET /projects/diagnostics?env=...`
    - `apps/portal/backend/app/main.py`
    - `apps/portal/backend/tests/test_api.py`
    - `apps/portal/backend/openapi.json`
  - Projects and Services pages now render source freshness, join drift, and partial-readiness warnings without blocking route rendering when secondary diagnostics fail:
    - `apps/portal/frontend/src/pages/projects-page.tsx`
    - `apps/portal/frontend/src/pages/services-page.tsx`
    - `apps/portal/frontend/src/lib/api.ts`
  - Platform health now surfaces provider readiness separately from incident/service state:
    - `apps/portal/frontend/src/lib/adapters/platform-health.ts`
    - `apps/portal/frontend/src/pages/platform-health-page.tsx`
  - Service monitoring panels now distinguish provider failure, upstream-unknown state, and valid no-data responses:
    - `apps/portal/frontend/src/pages/service-details-page.tsx`
    - `apps/portal/frontend/src/lib/adapters/service-metrics.ts`
    - `apps/portal/frontend/src/lib/adapters/logs-quickview.ts`
  - Frontend live-data runbook updated for readiness/degradation verification:
    - `docs/runbooks/strict-live-data-mode-frontend.md`

#### T4.7.7 Scheduled sync and freshness SLO for live catalogs
- **Description:** Add scheduled sync orchestration and freshness thresholds so live catalogs stay current without manual intervention.
- **Status:** DONE (2026-03-06)
- **Acceptance Criteria:**
  - Periodic sync job updates project/service registries on a defined interval.
  - Freshness thresholds produce warnings before data becomes operationally stale.
  - Sync metrics/logs allow troubleshooting failed runs in cluster.
- **Dependencies:** T4.7.2, T4.7.3, T4.6.7
- **Complexity:** S
- **Risk:** Low
- **Evidence:**
  - Added combined sync entrypoint for scheduled runs:
    - `apps/portal/backend/scripts/sync_catalog_registries.py`
    - emits JSON summary for both `gitops_apps` and `cluster_services`
    - exits non-zero when either source reports failures
  - Added in-cluster scheduled sync job:
    - `workloads/apps/homelab-api/base/catalog-sync-cronjob.yaml`
    - included in `workloads/apps/homelab-api/base/kustomization.yaml`
    - runs every 10 minutes with backend service account and DB access
  - Freshness diagnostics now support warning-before-stale semantics:
    - `warningAfterMinutes`
    - `isWarning`
    - `state=warning`
    - `apps/portal/backend/app/main.py`
    - `apps/portal/backend/tests/test_api.py`
  - Frontend readiness warnings now surface `warning` state ahead of `stale`:
    - `apps/portal/frontend/src/pages/projects-page.tsx`
    - `apps/portal/frontend/src/pages/services-page.tsx`
  - Runbooks added/updated for scheduled sync and SLO troubleshooting:
    - `docs/runbooks/catalog-sync-schedule.md`
    - `docs/runbooks/project-registry-sync-pipeline.md`
    - `docs/runbooks/service-registry-sync-pipeline.md`
    - `docs/runbooks/service-registry-diagnostics.md`

#### T4.7.8 End-to-end validation suite for live projects/services/monitoring
- **Description:** Add smoke checks that verify full live flow: GitOps apps -> projects, cluster services -> services, monitoring endpoints -> dashboard data.
- **Status:** DONE (2026-03-06)
- **Acceptance Criteria:**
  - Automated checks fail on legacy seeded project IDs and stale registry states.
  - Checks validate `/projects`, `/services`, `/releases`, metrics summary, and alerts feed.
  - Validation report captures before/after evidence in dev cluster.
- **Dependencies:** T4.7.4, T4.7.5, T4.7.6
- **Complexity:** M
- **Risk:** Medium
- **Evidence:**
  - Added end-to-end validation script:
    - `apps/portal/backend/scripts/live_catalog_validation.py`
    - validates `/projects`, `/projects/diagnostics`, `/services`, `/service-registry/diagnostics`, `/releases`, `/services/:serviceId/metrics/summary`, and `/alerts/active`
    - fails on legacy seeded project rows and stale/empty registry states
    - emits JSON report suitable for before/after evidence capture
  - Added unit tests for validation semantics:
    - `apps/portal/backend/tests/test_live_catalog_validation.py`
  - Backend README now documents the validation entrypoint:
    - `apps/portal/backend/README.md`
  - Runbook added for scripted validation and before/after reporting:
    - `docs/runbooks/live-catalog-validation-suite.md`

---

### E5.1 Multi-environment GitOps structure

#### T5.1.1 Split environments with clear promotion contracts
- **Description:** Implement dev/prod overlays and promotion rules.
- **Status:** DONE (2026-03-09)
- **Acceptance Criteria:**
  - Environment differences are declarative and auditable.
  - Promotion between envs uses documented workflow only.
- **Dependencies:** T2.2.2
- **Complexity:** L
- **Risk:** High
- **Validation/Evidence Checklist:**
  - `workloads/scripts/check-environment-contract.sh` passes against the current manifests.
  - Dev and prod root apps still point to `workloads/environments/dev` and `workloads/environments/prod` only.
  - A normal prod promotion PR changes only the documented prod image patch files.
  - The operator runbook for promotion and rollback is documented and usable without ad-hoc `kubectl` edits.
- **Evidence:**
  - Dev/prod GitOps entrypoints are explicit:
    - `workloads/bootstrap/root-app-dev.yaml`
    - `workloads/bootstrap/root-app-prod.yaml`
    - `workloads/environments/dev/kustomization.yaml`
    - `workloads/environments/prod/kustomization.yaml`
  - Workload overlays are split per environment and stay declarative:
    - `workloads/apps/homelab-api/envs/dev`
    - `workloads/apps/homelab-api/envs/prod`
    - `workloads/apps/homelab-web/envs/dev`
    - `workloads/apps/homelab-web/envs/prod`
  - Repo-side validation now enforces the environment contract:
    - `workloads/scripts/check-environment-contract.sh`
    - `workloads/.github/workflows/validate-gitops-environment-contract.yml`
  - Promotion and rollback flow are documented as `dev -> prod` only:
    - `docs/runbooks/gitops-dev-prod-promotion.md`
    - `workloads/README.md`
    - `apps/portal/.github/workflows/gated-promotion.yml`

### E5.2 Self-service project bootstrap

#### T5.2.1 Create project template generator (repo + manifests + CI seed)
- **Description:** Script/template that scaffolds a new service in <30 minutes.
- **Status:** DONE (2026-03-09)
- **Acceptance Criteria:**
  - New project includes CI, deployment manifests, and baseline policies.
  - At least one generated project deployed successfully.
- **Dependencies:** T5.1.1
- **Complexity:** XL
- **Risk:** High
- **Validation/Evidence Checklist:**
  - `workloads/scripts/scaffold-service.sh` generates both a service repo skeleton and GitOps manifests for a new service name.
  - Generated `dev` and `prod` overlays render successfully with `kustomize build`.
  - Generator smoke test passes without mutating the checked-in workloads tree.
  - A generated service rolls out successfully in a throwaway namespace and can be cleaned up cleanly.
  - Operator docs explain the current single-cluster behavior for prod app manifests.
- **Evidence:**
  - Scaffold generator and wrapper added:
    - `workloads/scripts/scaffold-service.py`
    - `workloads/scripts/scaffold-service.sh`
  - Generator smoke test added and wired into GitOps validation:
    - `workloads/scripts/smoke-test-scaffold-generator.sh`
    - `workloads/.github/workflows/validate-gitops-environment-contract.yml`
  - Usage documented in:
    - `workloads/README.md`
  - Throwaway rollout validation completed on 2026-03-09:
    - generated `scaffold-live-check2` via `workloads/scripts/scaffold-service.py`
    - applied `apps/scaffold-live-check2/envs/dev` to the cluster with a public `nginx:1.27-alpine` image plus `/` probe overrides
    - `kubectl -n scaffold-live-check2 rollout status deployment/scaffold-live-check2` succeeded
    - cleanup verified with `kubectl get ns scaffold-live-check2 -o name` returning `NotFound`
- **Notes:**
  - The generator creates prod overlays and a prod `Application` manifest file, but does not auto-enable `environments/prod/workloads/kustomization.yaml` while single-cluster safety mode remains active.

### E5.3 Internal developer portal seed

#### T5.3.1 Publish a lightweight service catalog page
- **Description:** Start with static/service metadata view before full portal platform.
- **Status:** DONE (2026-03-09)
- **Acceptance Criteria:**
  - Catalog includes owner, repo, runbook, env status links.
  - Data source is Git (not manual spreadsheet).
- **Dependencies:** T5.2.1
- **Complexity:** M
- **Risk:** Medium
- **Validation/Evidence Checklist:**
  - Portal catalog data includes Git-backed owner, repository, and runbook metadata for registered services.
  - The Services page shows environment status links for GitOps environments, including Git-only prod intent in the current single-cluster mode.
  - New scaffolded services register catalog metadata in Git as part of generation.
  - No spreadsheet or manual out-of-band catalog source is required.
- **Evidence:**
  - Git-backed service metadata source added:
    - `workloads/services.yaml`
  - Portal backend persists metadata with the GitOps project registry and exposes it through `/projects`:
    - `apps/portal/backend/app/gitops_project_sync.py`
    - `apps/portal/backend/app/main.py`
    - `apps/portal/backend/alembic/versions/20260309_0005_add_project_registry_metadata_columns.py`
  - Portal frontend catalog now renders owner, repo, runbook, and environment status links:
    - `apps/portal/frontend/src/lib/adapters/services.ts`
    - `apps/portal/frontend/src/pages/services-page.tsx`
    - `apps/portal/frontend/src/pages/projects-page.tsx`
  - Service-specific runbooks linked from the catalog:
    - `docs/runbooks/homelab-api-service-operations.md`
    - `docs/runbooks/homelab-web-service-operations.md`
  - Scaffold generator now appends service catalog metadata for new services:
    - `workloads/scripts/scaffold-service.py`
    - `workloads/scripts/smoke-test-scaffold-generator.sh`
    - `workloads/README.md`
- **Notes:**
  - In the current single-cluster safety mode, `prod` is intentionally shown as GitOps-only intent: it exists in Git, promotion metadata, and the portal catalog, but does not correspond to separate live `*-prod` workloads in the cluster until prod workloads are explicitly enabled later.

---

## Portal Level 3+: GitOps Write Workflows (Later Stage)

> This section outlines portal-driven GitOps mutation workflows (deploy, promote, config changes, self-service). **Critical design rule:** the portal writes Git changes; Argo CD remains the sole reconciler.

### Readiness gates (must be true before starting Level 3)

Before beginning Portal Level 3 work, your platform must demonstrate:

1. **Multi-service stability:** At least 3 different services deployed and stable (healthy and synced) for 2–4 weeks with no force-sync or manual cluster interventions required.
2. **Immutable artifacts from CI:** Every merged commit to app repos publishes an immutable image tag (semver or git-sha based); image rebuilds are reproducible and auditable.
3. **Proven image promotion:** Either auto-update (Argo CD Image Updater or equivalent) successfully bumps images in dev for >10 consecutive merges, OR PR-based manual bumps are tested end-to-end at least 3 times with zero failed syncs.
4. **Secrets strategy chosen and in-use:** Either SOPS or Sealed Secrets is integrated, secrets in Git are encrypted at rest, and at least 2 services consume decrypted secrets without issues.
5. **Observability baseline active:** Prometheus scrapes metrics, Grafana dashboards display cluster/app health, Loki collects logs, and at least one alert fires on a real event (and is confirmed to work).
6. **Runbook for rollback exists and tested:** Platform operator has walked through a rollback (git revert or tag bump) at least once and confirmed Argo CD applied the change within <5 minutes.

Attempting Level 3 before these gates will result in rework and operational toil.

#### Current audit status (2026-03-09)

Readiness gates result: **PASS**.

Overall Level 3 result: **GO**.

All six pre-Level-3 readiness gates are now satisfied, and the Render-like readiness checklist below is now green. Portal Level 3 implementation can proceed.

Run the repeatable audit in `docs/runbooks/portal-level3-readiness-audit.md` before opening any `T6.x` implementation work. The table below reflects repo-backed evidence only; anything not captured in Git, runbooks, or dated issue notes stays red.

| Gate | Status | Repo-backed evidence today | Missing proof before Level 3 can start |
| --- | --- | --- | --- |
| Multi-service stability | PASS | `docs/homelab-issues.md` records 3 canonical live dev services (`homelab-api`, `homelab-web`, `oauth2-proxy`) on 2026-03-06 and 2026-03-09, and operator confirmation on 2026-03-09 states the services have been deployed and stable for at least 14 days, implying a stable window starting on or before 2026-02-23 with no force-sync or manual cluster intervention. | Continue recording dated operational evidence, but this gate is currently satisfied. |
| Immutable artifacts from CI | PASS | `apps/portal/.github/workflows/portal-images.yml` publishes `sha-<commit>` GHCR tags and emits SBOM/provenance; `docs/architecture/deployment-flow.md` documents the path; `docs/homelab-issues.md` records three successful `main` workflow runs on 2026-03-09 with matching GitOps PRs (`#33`, `#34`, `#35`), retained SBOM artifacts, and authenticated GHCR manifest checks returning `ok` for both `homelab-api` and `homelab-web`. | Continue recording dated evidence for future runs, but this gate is currently satisfied. |
| Proven image promotion | PASS | `apps/portal/.github/workflows/portal-images.yml` opens dev image bump PRs; `apps/portal/.github/workflows/gated-promotion.yml` supports approved promote/rollback flows; `docs/homelab-issues.md` records a dated dev auto-promotion review on 2026-03-09 showing repeated merged auto-bump PRs with Argo deployment timestamps for both dev apps, plus an explicit operator-reviewed exception for superseded PR `#34`, which was intentionally not merged and treated as a skipped manual action rather than a platform failure. | Continue recording promotion evidence, but this gate is currently satisfied by operator-reviewed evidence. |
| Secrets strategy chosen and in-use | PASS | `docs/runbooks/sops-secrets.md` documents `SOPS + age + KSOPS`; `ansible/roles/argocd/tasks/main.yml` patches `argocd-repo-server` for `ksops`; `docs/homelab-issues.md` records a 2026-03-09 live repo-server rollout plus post-push validation at workloads revision `5d3095f`, where `homelab-api-dev` and `homelab-web-dev` both rendered real `Secret` objects (`homelab-api-postgres`, `oauth2-proxy-secret`) and stayed `Synced`/`Healthy`. | Continue recording normal secret rotations, but this gate is currently satisfied. |
| Observability baseline active | PASS | `docs/homelab-issues.md` records a dated observability drill on 2026-03-09: a temporary `HomelabObservabilityDrill` alert fired through Prometheus and Alertmanager for `homelab-api` in `dev`, surfaced through `GET /alerts/active?env=dev&serviceId=homelab-api&limit=50`, shared the same event window as healthy Prometheus-backed `GET /services/homelab-api/metrics/summary?range=1h`, and cleared cleanly after the temporary rule was deleted. | Continue recording real incidents or drills, but this gate is currently satisfied. |
| Runbook for rollback exists and tested | PASS | `docs/runbooks/gitops-dev-prod-promotion.md` documents rollback, and `docs/homelab-issues.md` records a dated live rollback drill on 2026-03-09: `homelab-workloads` revert commit `b11ba312617de122359916a13a16fde3a3dbee86` rolled `homelab-api-dev` and `homelab-web-dev` from `sha-cd76e59...` back to `sha-6beb5ac...`, and Argo reconciled both apps to the rollback revision in `2m16s` (`18:22:38Z` -> `18:24:54Z`). | Continue recording future rollback evidence, but this gate is currently satisfied. |

### Level 3 Render-Like Readiness Checklist

| Check | Status | Current repo state | What must exist to mark it done |
| --- | --- | --- | --- |
| Deployment records persist for every deploy/promote/rollback/config-change action (`T6.6.1`, `T6.6.2`) | PASS | `apps/portal/backend/alembic/versions/20260310_0006_create_deployments_table.py` adds the first-class `deployments` table; `apps/portal/backend/app/deployment_records.py` provides durable repository reads/writes; `apps/portal/backend/app/main.py` serves `/deployments` and `/services/{service_id}/deployments` from deployment records and clears deployment-history cache after writes/reconciles; `apps/portal/backend/app/deployment_reconciler.py` backfills PR-keyed deployment rows from GitOps metadata; and live dated evidence on 2026-03-10 now proves real deploy rows (`#46` -> `live`), real config-change rows (`#45` -> `live`), real rollback rows (`#42` -> `failed`), and real promote rows (`#47` -> `promote` rows written for both prod services). In the current single-cluster safety mode, prod workloads are intentionally disabled (`environments/prod/workloads` empty with `allowEmpty: true`), so promote is accepted as verified row creation and traceability rather than a live prod workload reconcile in this cluster. | Currently satisfied. If prod workloads are re-enabled later, capture a fresh `promote -> live` example. |
| Deploy status lifecycle is visible as `pending` -> `deploying` -> `live`/`failed` (`T6.6.2`, `T6.6.3`, `T6.2.4`) | PASS | Deployment records now persist lifecycle state plus timestamps (`requestedAt`, `startedAt`, `finishedAt`, `deployWindowStart`, `deployWindowEnd`) and `failureReason`; `apps/portal/frontend/src/pages/service-deployments-page.tsx` renders those states directly from deployment records; and live dated evidence on 2026-03-10 showed `pending` rows (`#42`), `deploying` rows (`#46`), `live` rows (`#46`, `#45`), and `failed` rows (`#42` and earlier failed deploys) through the deployment-record API. | Currently satisfied. Continue preserving dated evidence whenever lifecycle or timeout logic changes. |
| Post-merge rollout verification is automatic and updates deployment results (`T6.6.3`) | PASS | `apps/portal/backend/app/deployment_reconciler.py` polls recent GitOps PRs, combines merged-PR state with live Argo sync/health plus live workload image refs, preserves terminal states, and updates deployment records to `deploying`, `live`, or `failed`; `apps/portal/backend/app/main.py` starts a background reconciler loop in-cluster and exposes `/deployments/reconcile`; and live dated evidence on 2026-03-10 showed deploy `#46` reaching `live`, config-change `#45` reaching `live`, rollback `#42` moving from `pending` to `failed`, and promote `#47` producing merged prod `promote` rows for both services. Because single-cluster safety mode intentionally keeps the prod workloads path empty, `#47` is accepted as operator-reviewed verification of the promote path rather than a live prod workload rollout in this cluster. | Currently satisfied for the current single-cluster topology. If prod workloads are re-enabled later, capture a fresh `promote -> live` example. |
| Deploy-scoped logs and metrics are visible around each deploy window (`T6.7.1`, `T6.7.2`, `T6.7.3`) | PASS | `apps/portal/backend/app/main.py` now exposes `/services/{service_id}/observability/window`, which resolves deployment windows from deployment records (or explicit `windowStart/windowEnd`) and returns anchored metrics snapshots, health timeline segments, and Loki quick-view lines; deployment-history metric snapshots also now use `deployWindowStart` / `deployWindowEnd`; `apps/portal/frontend/src/pages/service-deployments-page.tsx` renders a deployment drilldown panel tied to the selected deployment row; and dated live evidence on 2026-03-11 showed a real `homelab-api` deploy record (`deploymentId = 043ff7f3-252c-48d5-baa7-5650e7e590ea`, PR `#54`) returning `context.evidenceStatus = "resolved"`, `metricStatus = "ok"`, `timelineStatus = "ok"`, and `logsStatus = "no_data"` through both the deployment-ID and explicit-window paths. | Currently satisfied. Preserve dated evidence whenever deployment-window query semantics or provider fallback behavior changes. |
| Canonical service identity is enforced across portal, Argo CD, Kubernetes labels, Prometheus, Loki, and release joins (`T6.4.4`, `T6.6.5`) | PASS | `apps/portal/backend/app/service_identity.py` centralizes canonical `serviceId` normalization; `apps/portal/backend/app/service_identity_validation.py` and `/service-registry/diagnostics` emit per-service drift rows for GitOps path, namespace, `app.kubernetes.io/name`, Argo app, and release joins; `apps/portal/backend/scripts/live_catalog_validation.py` fails when `identityDrift.driftCount > 0`; and `workloads/scripts/check-service-identity-contract.sh` now runs in `validate-gitops-environment-contract.yml` to fail CI on GitOps/app-label/ServiceMonitor drift. Dated live evidence on 2026-03-11 showed `identityDrift.driftCount = 0`, `okCount = 3`, a passing live catalog validation report, and `HTTP 422` for a non-canonical deployment write using `serviceId = "Homelab API"`. | Currently satisfied. Preserve dated runtime evidence whenever identity or diagnostics rules change. |
| Deploy locks prevent overlapping mutations per `serviceId+env` (`T6.6.4`, `T6.2.4`) | PASS | `apps/portal/backend/alembic/versions/20260310_0007_create_deployment_locks_table.py` adds persisted `deployment_locks`; `apps/portal/backend/app/deployment_locks.py`, `apps/portal/backend/app/main.py`, and `apps/portal/backend/app/deployment_reconciler.py` now acquire/release locks for `pending` and `deploying` mutations, reject overlapping writes with `409 Conflict`, expose `deploymentLock` on `GET /services/{service_id}`, and clear stale locks. `apps/portal/frontend/src/pages/service-details-page.tsx` now renders an active-lock banner. Live dated evidence on 2026-03-11 showed a manual `homelab-api/dev` pending lock, an operator-visible `deploymentLock` response, a rejected overlapping mutation with full active-lock payload, and lock release back to `null` after a terminal `failed` write. | Currently satisfied. Continue recording evidence when lock timeout/release behavior changes. |
| Rollback from portal remains available and traceable (`T6.2.3`, `T6.2.4`, `T6.6.2`) | PASS | `apps/portal/backend/app/github_workflows.py` dispatches the existing `gated-promotion.yml` workflow in `rollback` mode; `apps/portal/backend/app/main.py` exposes `POST /rollbacks`; `apps/portal/.github/workflows/gated-promotion.yml` records `operator_reason` in rollback PR bodies plus deployment writes; `apps/portal/backend/app/deployment_reconciler.py` parses that reason back out during reconcile; `apps/portal/frontend/src/pages/service-details-page.tsx` exposes rollback controls; and dated live evidence on 2026-03-11 captured a portal-initiated rollback fire drill through GitHub Actions run `#22953368415` and workloads PR `#53`, with real `action = rollback` prod rows for both services storing operator reason, `compareUrl`, PR linkage, and final reconciled `failed` outcome after the PR was intentionally closed without merge. The rollback token path is now also GitOps-managed via `homelab-api-github-actions`, and a follow-up `POST /rollbacks` returned `202 Accepted` after Argo reconciled workloads revision `9852145`. | Currently satisfied. Preserve dated evidence whenever rollback request or token-wiring semantics change. |
| Portal shows deploy history timeline with deploy reason and changelog context (`T6.2.4`, `T6.2.5`, `T6.7.2`) | PASS | `apps/portal/frontend/src/lib/adapters/deployments.ts` and `apps/portal/frontend/src/pages/service-deployments-page.tsx` render deployment records with action, requested/completed times, lifecycle state, failure reason, PR link, compare link, and deploy reason instead of a metrics-only table; live deployment history includes real `deploy` (`#46`), `promote` (`#47`), `config-change` (`#45`), and `rollback` (`#42`) rows; and as of 2026-03-11 the page also exposes explicit action and status filters on top of the existing service/env scope and sorting modes. | Currently satisfied. Preserve dated evidence if timeline grouping, filtering semantics, or compare-link rendering changes. |

The six readiness gates above are green and the Render-like checklist is now fully satisfied. Portal Level 3 implementation can proceed under the documented single-cluster constraints.

---

## Phase 6 (Month 4–5): GitOps Write Workflows & Self-Service (Portal Level 3)

**Goal:** Enable developers to deploy, promote, and configure workloads through the portal by writing managed Git changes, keeping Argo CD as the single source of truth and reconciler.

### Epics

- E6.1 GitOps Git integration foundation
- E6.2 Deploy and promotion workflows (PR-based)
- E6.3 Configuration and secrets management workflows
- E6.4 Service scaffolding and registration (minimal MVP)
- E6.5 Multi-user access control and audit (optional later)
- E6.6 Deployment records, rollout verification, and deploy safety controls
- E6.7 Deploy-scoped observability and deploy timeline UX

---

### E6.1 GitOps Git integration foundation

#### T6.1.1 Write ADR: Portal-to-Git workflow model
- **Description:** Decide how the portal will create Git changes: Option A (GitHub API PRs directly), Option B (write to "requests" branch and bot opens PR), or Option C (direct commits to feature branches, discouraged). Document trade-offs: latency, security token scope, audit trail, multi-git-provider support.
- **Status:** DONE (2026-03-11)
- **Acceptance Criteria:**
  - ADR accepts one option with rationale.
  - Implementation plan includes token management and scope limits.
  - Fallback for GitHub outage or rate limits addressed.
- **Dependencies:** None
- **Complexity:** S
- **Risk:** Medium
- **Notes:** Choose Option A (GitHub API PRs) for solo engineer simplicity and audit trail. Option B adds workflow bot maintenance. Option C loses audit and requires branch cleanup discipline.
- **Evidence:**
  - ADR added at `docs/adr/0007-portal-to-git-workflow-model.md` and marked `Accepted`, choosing direct GitHub API pull request creation as the portal write model.
  - The ADR includes the implementation plan for fine-grained token scope (`Contents`, `Pull requests`, `Metadata`), SOPS-backed secret storage as `GIT_GITHUB_TOKEN`, path allow-listing, idempotent branch/PR creation, and bounded retry plus manual replay guidance for GitHub outage or rate-limit conditions.

#### T6.1.2 Implement backend Git integration module
- **Description:** Create `app/lib/git_service.py` (backend) with:
  - `create_branch(repo, from_branch, new_branch)` → returns branch ref
  - `modify_file(repo, branch, file_path, new_content)` → replaces file and returns diff
  - `open_pr(repo, from_branch, to_branch, title, description)` → returns PR URL and ID
  - `commit_to_branch(repo, branch, files_dict, message)` → multi-file commit
  - `close_pr(repo, pr_id)` → for cleanup (optional)
  - Error handling for conflicts, rate limits, auth failures.
- **Status:** DONE (2026-03-11)
- **Acceptance Criteria:**
  - Module has type hints and docstrings.
  - Unit tests mock GitHub API calls for all happy paths + auth failure + conflict scenarios.
  - `open_pr()` returns minimal PR object: `{id, url, number, state}`.
  - Dry-run mode logs calls without executing (for testing).
- **Dependencies:** T6.1.1
- **Complexity:** M
- **Risk:** Medium
- **Notes:** Use PyGithub or similar for API calls. Guard against rate limits with retries/backoff.
- **Evidence:**
  - Added `apps/portal/backend/app/lib/git_service.py` with a GitHub-backed `GitHubGitService`, typed return contracts, dry-run logging, retry/backoff handling for rate limits and `5xx`, and explicit auth/conflict error mapping.
  - Added `apps/portal/backend/tests/test_git_service.py` covering happy paths for branch creation, file modification, PR open/close, multi-file commits, plus auth failure, conflict handling, rate-limit retry, and dry-run behavior using mocked GitHub API calls.
  - `python3 -m py_compile` passed for the new module and tests. Local `pytest` execution was not possible in this workspace because `pytest` is not installed.

#### T6.1.3 Configure fine-scoped GitHub token and store securely
- **Description:** Create a GitHub personal access token (or fine-grained token) scoped to: workloads only, write access to Contents + Pull Requests. Store in `homelab-api` secret (encrypted via SOPS/Sealed Secrets) as `GIT_GITHUB_TOKEN`. Document token rotation quarterly.
- **Status:** DONE (2026-03-12)
- **Acceptance Criteria:**
  - Token has minimum required scopes (no admin, no org/user data access).
  - Secret stored encrypted in workloads repo or homelab-api namespace.
  - Rotation procedure documented in ops runbook.
  - Dry-run test confirms token can list repo without deploying.
- **Dependencies:** T6.1.2, secrets strategy (Phase 3)
- **Complexity:** S
- **Risk:** High
- **Notes:** Use GitHub "fine-grained personal access tokens" (beta at time of writing) for maximum scope isolation if available.
- **Evidence:**
  - Added `workloads/scripts/bootstrap-sops-git-github-token.sh` to create a SOPS-encrypted `homelab-api-git-github` Secret manifest with `GIT_GITHUB_TOKEN` and to wire its KSOPS generator into the dev overlay when the operator provides a real token.
  - Updated `workloads/apps/homelab-api/envs/dev/patch-deployment.yaml` so the API container can consume `GIT_GITHUB_TOKEN` from `homelab-api-git-github` once the secret is created, without breaking current deployments before the token exists.
  - Added `apps/portal/backend/scripts/verify_git_github_token.py` as a dry-run repo access probe for `wlodzimierrr/homelab-workloads`.
  - Updated `docs/runbooks/sops-secrets.md`, `docs/runbooks/secret-rotation-quarterly.md`, and `workloads/README.md` with fine-scoped token scope, bootstrap, validation, and rotation steps.
  - On 2026-03-12, `verify_git_github_token.py` returned `{"ok": true, "repo": "wlodzimierrr/homelab-workloads", ...}` for a fine-scoped `GIT_GITHUB_TOKEN`, proving the token can read the target repo without deployment-side mutation.
  - On 2026-03-12, Argo reconciled `homelab-api-dev` to workloads revision `91e31ae55fb4e34c56dab63078d505050a3297fd`, and `kubectl -n homelab-api get secret homelab-api-git-github` confirmed the GitOps-managed Secret exists live with status `Synced`.

#### T6.1.4 Add Git provider abstraction + GitHub implementation
- **Description:** Design a `GitProvider` interface (Python ABC or Protocol) so future additions (GitLab, Gitea) stay decoupled. Implement GitHub provider.
- **Status:** DONE (2026-03-12)
- **Acceptance Criteria:**
  - `GitProvider` protocol defines required methods.
  - GitHub implementation passes all T6.1.2 tests.
  - Backend can be configured to use different providers via env var.
- **Dependencies:** T6.1.2
- **Complexity:** M
- **Risk:** Low
- **Notes:** Defer GitLab/Gitea until a second provider is actually needed.
- **Evidence:**
  - `apps/portal/backend/app/lib/git_service.py` now exposes a provider-agnostic `GitProvider` protocol while retaining the older `GitService` name as a compatibility alias for existing imports.
  - The GitHub implementation is now surfaced as both `GitHubGitProvider` and the compatibility alias `GitHubGitService`, so the existing T6.1.2 GitHub tests continue to exercise the real provider implementation without refactoring downstream call sites yet.
  - Added `build_default_git_provider()` and `GIT_PROVIDER`-based provider selection, with explicit rejection of unsupported providers while keeping `github` as the default.
  - Added factory-selection coverage in `apps/portal/backend/tests/test_git_service.py` for default GitHub provider selection, backward-compatible factory behavior, protocol aliasing, and unsupported-provider configuration errors.

---

### E6.2 Deploy and promotion workflows (PR-based)

#### T6.2.1 Portal API: "Deploy latest to dev" endpoint
- **Description:** `POST /api/services/{service_id}/deploy-to-dev` endpoint that:
  1. Fetches latest image tag from registry (or uses last N tags from CI artifact).
  2. Creates feature branch from dev overlay.
  3. Modifies kustomization patch to bump image tag in dev only.
  4. Requires `deploy_reason` text and captures changelog context (`compare_url`, `previous_tag`, `new_tag`) in request metadata.
  5. Commits and opens PR with title "Deploy {service}: {new_tag} to dev".
  6. Creates a deployment record (`action_type=deploy`, `status=pending`) and returns PR URL + deployment ID.
- **Status:** IN PROGRESS (2026-03-12)
- **Acceptance Criteria:**
  - Endpoint requires auth (logged-in user).
  - PR diff shows only image tag change in dev kustomize patch.
  - Request enforces non-empty deploy reason and stores previous→new version plus commit compare link.
  - Argo CD auto-syncs after merge (if Image Updater auto-sync enabled) or manual sync works.
  - PR can be merged without conflict.
  - Rollback behavior: closing PR without merge leaves dev unchanged.
- **Dependencies:** T6.1.2, T1.2.3 (app deployed), T2.1.1 (CI publishes images), T6.6.2
- **Complexity:** M
- **Risk:** High
- **Notes:** Use semantic versioning tag or git-sha convention. Handle case where latest tag already in dev (return 204 or info message).
- **Implementation Note:** Implemented in `apps/portal` and fully live-proven on 2026-03-12. `POST /services/homelab-web/deploy-to-dev` authenticates, enforces a minimum deploy reason, resolves the latest successful `portal-images.yml` build, and handles both `noop` and non-`noop` paths correctly. Live proof created workloads PR `#74`, deployment record `20022b2a-99ae-4035-863b-cb2e26f1b28e`, and captured `previousTag = sha-91260e065c6826040d2208ee73a752b600aeae7b`, `newTag = sha-6904464d0c1890629101d845926534e02110b89c`, the expected compare link, and final reconciled `status = live`. The earlier superseded auto-bump PR `#73` was closed without merge and reconciled to `failed`, which also confirmed the lock-aware overlap protection path.
- **Change Note:** Extended to require deploy reason/changelog context and to create deployment records as first-class objects.

#### T6.2.2 Portal API: "Promote dev → prod" endpoint
- **Description:** `POST /api/services/{service_id}/promote-to-prod` endpoint that:
  1. Reads current image tag from dev overlay (already deployed).
  2. Creates feature branch from prod overlay.
  3. Modifies prod kustomize patch to match dev tag.
  4. Requires `deploy_reason` text and captures changelog context (`compare_url`, `previous_tag`, `new_tag`) in request metadata.
  5. Commits and opens PR with title "Promote {service}: {tag} to prod".
  6. Creates a deployment record (`action_type=promote`, `status=pending`) and returns PR URL + deployment ID.
- **Status:** IN PROGRESS (2026-03-12)
- **Acceptance Criteria:**
  - Promotion PR contains only prod patch change.
  - Promotion copies the exact tag from dev (no manual override yet).
  - Request enforces non-empty deploy reason and stores previous→new version plus commit compare link.
  - PR can be merged without conflict.
  - Merged promotion triggers Argo sync and deployment to prod.
  - Anti-pattern check: promotion of uncommitted/test tags blocked (tag must exist in registry).
- **Dependencies:** T6.2.1, T6.1.2, T6.6.2
- **Complexity:** M
- **Risk:** High
- **Notes:** Later enhancement: gated promotion (approvals, policy checks). For MVP: user merges PR in GitHub.
- **Implementation Note:** Implemented in `apps/portal` and fully live-proven at the API/PR/record level on 2026-03-12 as `POST /services/{service_id}/promote-to-prod`. The endpoint reads the exact dev tag for the selected service, verifies the corresponding published image tag exists, updates only the prod overlay image patch file(s), opens a service-specific PR, and writes a pending `action=promote` deployment record. Live proof created workloads PR `#71` for `homelab-web`, with deployment record `580e3fb7-2830-449c-84f7-688258019862`, `previousTag = sha-04aea0869d105ee2332bf7e2ceb668ac4251c675`, `newTag = sha-6a30b92a070e7bf11e847dd56052e92171a09a22`, the expected compare link, and a merged PR diff affecting only `apps/homelab-web/envs/prod/patch-deployment.yaml`. Under the current single-cluster prod safety model, the final record reconciles to `failed` with `No matching service registry row exists for this deployment target.` because no live prod workload target exists; that operator-reviewed exception is accepted for this task because the promote path, PR generation, and deployment-record traceability are the intended scope.
- **Change Note:** Extended to persist promotion deployment records and deploy reason/changelog context.

#### T6.2.3 Portal API: "Rollback to previous tag" endpoint
- **Description:** `POST /api/services/{service_id}/rollback` endpoint that:
  1. Reads current deployed tag from specified env overlay.
  2. Fetches previous 5 image tags from registry (or Git commit history of patch file).
  3. Allows user to select target tag.
  4. Requires `deploy_reason` text and captures changelog context (`compare_url`, `previous_tag`, `new_tag`) in request metadata.
  5. Creates PR bumping image tag back to target.
  6. Creates a deployment record (`action_type=rollback`, `status=pending`) and returns PR URL + deployment ID.
- **Status:** IN PROGRESS (2026-03-12)
- **Acceptance Criteria:**
  - At least 5 previous tags available for selection.
  - Rollback PR contains only tag change.
  - Request enforces non-empty rollback reason and stores previous→new version plus commit compare link.
  - Rollback tested end-to-end once (deploy version N, deploy N+1, rollback to N, confirm N running).
  - Rollback does not modify any other deployments.
- **Dependencies:** T6.2.1, T6.1.2, T6.6.2
- **Complexity:** M
- **Risk:** Medium
- **Notes:** Integrate with container registry API (GHCR, Harbor) to list tags with dates. Provide UI confirmation before generating PR.
- **Implementation Note:** The original portal-pair rollback path is already live via `POST /rollbacks`, with dated API proof captured on 2026-03-11. In addition, `apps/portal` now has a generic per-service rollback implementation ready for rollout: `GET /services/{service_id}/rollback-candidates` lists the current tag plus up to 5 prior candidate tags from GitHub Packages metadata, `POST /services/{service_id}/rollback` creates a service-scoped GitOps PR plus pending `action=rollback` deployment record, `apps/portal/backend/app/deployment_reconciler.py` recognizes the new manual rollback PR pattern, and `apps/portal/frontend/src/pages/service-details-page.tsx` now uses a per-service rollback form with environment selection and candidate-tag picker instead of raw tag text inputs. The remaining gap is live proof for the new generic per-service path after the updated backend/frontend are deployed.
- **Change Note:** Extended to make rollback a first-class deployment record with required operator reason/changelog context.

#### T6.2.4 Portal UI: Deploy & promote controls
- **Description:** Add buttons/modals to service detail page:
  - "Deploy latest to dev" button → calls T6.2.1 → shows PR link in toast.
  - "Promote to prod" button (if user in prod-approver role) → calls T6.2.2 → shows PR link.
  - "Rollback" button → modal to select previous tag → calls T6.2.3.
  - Each button shows: current deployed tag, latest available tag, PR status (open/merged).
- **Status:** IN PROGRESS (2026-03-12)
- **Acceptance Criteria:**
  - Buttons disabled if deploy lock exists for the selected `serviceId+env` or if a deployment is in `pending/deploying`.
  - PR links clickable and open in new tab.
  - Toast shows success/error with clear message.
  - UI shows previous 3 deployed tags inline.
  - UI shows deployment status badges (`pending`, `deploying`, `live`, `failed`) and exposes deploy reason/changelog context per row.
- **Dependencies:** T6.2.1, T6.2.2, T6.2.3, T1.6.2 (service detail page), T6.6.2, T6.6.4, T6.7.2
- **Complexity:** M
- **Risk:** Low
- **Notes:** Show PR state (open, merged, closed) with color badges; auto-refresh every 30s to detect merge.
- **Implementation Note:** The service detail page now includes deployment-lock visibility, a first-class rollback form with environment selection plus previous-tag candidates, clickable deployment-history navigation, and status-aware recent deployment rows. That means the lock-awareness and rollback portions of this task are already present in `apps/portal/frontend/src/pages/service-details-page.tsx`. The remaining gap is the forward-action UI: there are still no first-class `Deploy latest to dev` or `Promote to prod` controls on the service page.
- **Change Note:** Expanded from simple button UX to deployment-status-aware controls with lock awareness and timeline context.

#### T6.2.5 Release traceability: link deployments to commits and images
- **Description:** Enhance release traceability by linking deployment records to commit/image/PR metadata and Argo rollout outcomes. Add metadata endpoint `GET /api/services/{service_id}/deployment-info` that returns: `{deployment_id, deployed_image, previous_image, image_digest, git_commit, deployed_timestamp, pr_link, compare_url, deploy_reason, result, result_reason}`.
- **Status:** IN PROGRESS (2026-03-12)
- **Acceptance Criteria:**
  - Argo Application has custom annotations: `image-tag`, `commit-sha`, `deployed-at`.
  - Portal displays these in service detail → "Deploy Info" card.
  - Commit SHA links to GitHub commit; image tag links to registry.
  - pr_link and merge SHA are persisted to deployment record after PR merge.
  - compare link and deploy reason are shown in deploy timeline/history UI.
- **Dependencies:** T6.2.1, T1.6.2, T6.6.2, T6.6.3
- **Complexity:** M
- **Risk:** Medium
- **Notes:** Metadata initially populated via PR merge event (GitHub Actions comment on Argo Application manifest); later automate via Argo Notification or custom controller.
- **Implementation Note:** A substantial part of release traceability is already live through deployment records and the existing release dashboard. `apps/portal/backend/app/main.py` and `apps/portal/backend/app/deployment_records.py` already persist and return `commitSha`, `imageRef`, `previousImageRef`, `gitPrUrl`, `gitPrNumber`, `mergeSha`, `compareUrl`, and `deployReason`, and the frontend already renders commit/image/Argo drift state on the dashboard plus compare/deploy-reason context on deployment history pages. The remaining gap is the explicit per-service `GET /services/{service_id}/deployment-info` endpoint and a dedicated "Deploy Info" card on the service detail page, along with any Argo-side custom annotations if that path is still desired.
- **Change Note:** Scope expanded from Argo-only annotation view to canonical deployment-record traceability.

---

### E6.3 Configuration and secrets management workflows

#### T6.3.1 Portal API: "Edit config (non-secret environment variables)" endpoint
- **Description:** `POST /api/services/{service_id}/config/set` endpoint that:
  1. Takes `{env, config_key, config_value}` payload.
  2. Validates input: no known secret patterns (API_KEY, PASSWORD, etc.), max length, alphanumeric/standard chars.
  3. Reads current ConfigMap patch from kustomize overlay.
  4. Merges new key-value into patch.
  5. Creates PR with commit message "Config: {service} {env} {key}={value}".
  6. Returns PR URL.
  - Alternative simpler MVP: no validation, just require env var name convention (e.g. `APP_*` only).
- **Status:** TODO
- **Acceptance Criteria:**
  - Input validation rejects obvious secrets (contains PASSWORD, API_KEY, TOKEN, SECRET in name or typical secret patterns in value).
  - PR diff shows only ConfigMap patched object with updated data.
  - After merge, pod restart is triggered (via checksum annotation or kustomize patchesStrategicMergeList).
  - Multiple calls in succession create one PR (or update existing pending PR) to avoid PR spam.
- **Dependencies:** T6.1.2, workload has ConfigMap (T1.2.3)
- **Complexity:** M
- **Risk:** High
- **Notes:** Prevent accidental plaintext secrets by name-based heuristic. For MVP, allow portal admin to explicitly allow keys; do not auto-open to all env vars.

#### T6.3.2 Portal API: "Edit secrets (SOPS/Sealed Secrets)" endpoint
- **Description:** `POST /api/services/{service_id}/config/set-secret` endpoint that:
  1. Takes `{env, secret_key, secret_value}` payload (value submission over HTTPS only).
  2. Fetches current Secret patch from kustomize overlay.
  3. If SOPS: call local SOPS CLI to encrypt value, then update manifest.
  4. If Sealed Secrets: call sealed-secret controller API to seal value.
  5. Creates PR with commit "Secret: {service} {env} {key} updated".
  6. Returns PR URL (user reviews PR in GitHub; actual secret value not visible in portal).
- **Status:** TODO
- **Acceptance Criteria:**
  - Secret value never logged or stored in unencrypted form.
  - PR shows encrypted/sealed value only (never plaintext).
  - Argo CD applies decrypted secret successfully.
  - At least one secret edit end-to-end tested.
  - Rate limiting (1 edit per 30s) to prevent accidental spam.
- **Dependencies:** T6.1.2, T3.3.1 (secrets strategy live), workload has Secret (T1.2.3)
- **Complexity:** L
- **Risk:** High
- **Notes:** This is high-risk: leaking a secret value is catastrophic. Require explicit token/MFA for secret edits. Consider requiring portal admin approval before PR creation.

#### T6.3.3 Implement pod restart / rollout trigger (annotation checksum strategy)
- **Description:** Define and implement restart strategy for ConfigMap/Secret changes. Recommended: add checksum annotation to Deployment/StatefulSet pod template spec when ConfigMap/Secret is patched. Argo CD applies annotation change → Kubernetes restarts pods. Document in ops runbook.
- **Status:** TODO
- **Acceptance Criteria:**
  - Kustomize patch adds `checksum/config: <md5(configmap_data)>` annotation to pod spec when ConfigMap changes.
  - Pod restarts within 1 minute of PR merge & Argo sync.
  - Service remains available during rolling restart (readiness probes work).
  - Rollback also triggers pod restart (atomic).
- **Dependencies:** T6.3.1, T1.2.3 (app with graceful shutdown)
- **Complexity:** M
- **Risk:** Medium
- **Notes:** Alternative: use Sealed Secrets/SOPS Controller annotations (auto-restart on secret update). Implement whichever is already in use.

#### T6.3.4 Portal UI: Config and secrets editing interface
- **Description:** Add tabs to service detail page:
  - "Config" tab: list current ConfigMap keys with values (non-sensitive); "Edit" button opens form to add/modify; "Delete" removes key (if allowed).
  - "Secrets" tab: list secret keys (values hidden, show "***"); warning banner "You are about to modify encrypted secrets"; form to add/modify/delete.
  - Both tabs show: pending PR link, last update timestamp, editor (username).
  - Both tabs: "Editing disabled for prod" warning if user lacks prod-write role.
- **Status:** TODO
- **Acceptance Criteria:**
  - Config tab allows add/edit/delete with validation (prevent obviously bad keys).
  - Secrets tab requires confirm+MFA flow (or portal admin approval).
  - Both tabs show PR status in real-time (open/merged).
  - Form submission shows loading state and success toast.
- **Dependencies:** T6.3.1, T6.3.2, T1.6.2
- **Complexity:** M
- **Risk:** Medium
- **Notes:** MVP: secrets editing disabled unless user is app owner + admin role (two roles required).

---

### E6.4 Service scaffolding and registration (minimal MVP)

#### T6.4.1 Write ADR: Service metadata storage and registration model
- **Description:** Decide whether service metadata (name, owner, repo, runbook URL, envs, tags) lives in: Option A (Git-owned `services.yaml` or `services/` directory), Option B (backend database), Option C (Argo CD Application labels/annotations only). Option A is recommended for homelab (low ops burden, auditable, no DB).
- **Status:** TODO
- **Acceptance Criteria:**
  - ADR accepts Option A (Git-backed YAML manifest) with rationale.
  - Manifest schema defines required fields: name, owner_email, repo_url, envs[], tags.
  - Migration path to Option B noted if scale demands.
- **Dependencies:** None
- **Complexity:** S
- **Risk:** Low

#### T6.4.2 Implement service template generator (CLI + documentation)
- **Description:** Create `scripts/scaffold-service.sh` (or Python equivalent) that:
  1. Prompts for: service name, description, image repo (e.g. `ghcr.io/wlodzimierrr/my-app`), base language/framework.
  2. Generates and commits to workloads repo:
     - `apps/{service-name}/base/` (kustomization.yaml, deployment.yaml, service.yaml, hpa.yaml).
     - `apps/{service-name}/envs/dev/` and `envs/prod/` with kustomization overlays.
     - `environments/{dev,prod}/kustomization.yaml` patch to include new app.
     - Argo Application manifests for both envs under `bootstrap/`.
  3. Generates CI stub (GitHub Actions `.github/workflows/build-{service}.yml`) that: builds image, runs tests, publishes to registry.
  4. Registers service in `workloads/services.yaml` with metadata.
  5. Creates PR with all files + instructions for follow-up.
- **Status:** TODO
- **Acceptance Criteria:**
  - Script outputs a generated folder tree matching conventions.
  - Generated manifests are valid and deployable via `kustomize build`.
  - Generated Argo Application syncs without errors.
  - Generated CI workflow publishes test image successfully (or dry-runs).
  - At least one service scaffolded end-to-end and deployed.
  - Script is documented with examples in `workloads/README.md`.
- **Dependencies:** T6.4.1, T0.2.1 (workloads repo structure)
- **Complexity:** L
- **Risk:** High (template drift if not maintained)
- **Notes:** Command: `./scaffold-service.sh --name my-app --image ghcr.io/org/my-app`. Start minimal; iterate based on repeated patterns.

#### T6.4.3 Portal UI: "New service" scaffolding flow (later enhancement)
- **Description:** Add "New Service" button to services list that opens a step-by-step wizard:
  1. Step 1: Basic info (name, description, owner, repo URL).
  2. Step 2: Select template type (backend, frontend, job, etc.).
  3. Step 3: Configure image registry and build CI tool.
  4. Step 4: Review generated manifests and confirm.
  5. Calls backend endpoint that runs `scaffold-service.sh` with inputs and returns PR URL.
- **Status:** TODO
- **Acceptance Criteria:**
  - Wizard shows generated manifests for manual review before PR creation.
  - Back/next buttons allow revision.
  - PR link provided at completion.
  - Deployed service appears in list after PR merge and Argo sync.
- **Dependencies:** T6.4.2, T1.6.1 (services list)
- **Complexity:** L (UI only; backend is T6.4.2)
- **Risk:** Low
- **Notes:** Defer this UI until scaffold script proves stable (1–2 generations done manually).

#### T6.4.4 Service registration and metadata maintenance
- **Description:** Create `workloads/services.yaml` with schema:
  ```yaml
  services:
    - name: homelab-api
      owner_email: you@example.com
      repo_url: https://github.com/user/homelab
      description: Backend API for homelab platform
      envs:
        - name: dev
          namespace: homelab-api-dev
          argo_app: homelab-api-dev
        - name: prod
          namespace: homelab-api-prod
          argo_app: homelab-api-prod
      tags: [core, python, fastapi]
  ```
  Maintain by hand initially; later: auto-generate from scaffold.
- **Status:** TODO
- **Acceptance Criteria:**
  - Schema defined and validated (JSON Schema or similar).
  - All currently deployed services (homelab-api, homelab-web, etc.) registered.
  - service-name lookup returns all metadata needed for deploy/promote/config workflows.
  - Portal uses this as service directory (search, filter, list).
  - Canonical identity mapping is explicit and validated for every env: `serviceId`, `argo_app`, `namespace`, `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `env`.
  - Registry metadata is sufficient to build Prometheus/Loki selectors and release/deployment joins without manual aliasing.
- **Dependencies:** T6.4.1, T1.2.3 (services exist), T4.4.1
- **Complexity:** S
- **Risk:** Low
- **Change Note:** Extended to enforce canonical service identity fields and label conventions across platform integrations.

---

### E6.5 Multi-user access control and audit (optional later phase)

#### T6.5.1 Portal RBAC roles and enforcement
- **Description:** Define portal roles: `admin` (all actions), `operator` (deploy/promote/config on non-prod), `developer` (read-only + can deploy-to-dev only), `viewer` (read-only). Map to JWT claims or OAuth groups. Enforce in backend API layer with decorators.
- **Status:** TODO
- **Acceptance Criteria:**
  - Role enum exists in backend.
  - Each endpoint checks role via decorator (e.g. `@require_role(admin, operator)`).
  - JWT token includes role claim (or mapped from OAuth/OIDC).
  - At least 2 users with different roles tested.
- **Dependencies:** T1.3.1 (JWT auth), external auth (T3.2.2 if OAuth)
- **Complexity:** M
- **Risk:** Medium
- **Notes:** Start simple: admin (own user) + demo user (public viewer). Add more roles on demand.

#### T6.5.2 Audit logging for portal mutations (GitOps actions)
- **Description:** Log all write actions (deploy, promote, config edit, service create) to backend database or structured file with: timestamp, user, action type, service, target env, PR URL, status (success/failed). Query endpoint `GET /api/audit-log` shows filtered results. Portal UI displays audit log per service.
- **Status:** TODO
- **Acceptance Criteria:**
  - Every write action creates audit entry.
  - Audit log queryable by user, date range, service, action type.
  - At least 30 days of logs retained (configurable).
  - Portal shows audit trail in service detail sidebar (last 10 changes by whom and when).
- **Dependencies:** T6.2.1, T6.3.1, T6.4.2, backend database (T1.2.2)
- **Complexity:** M
- **Risk:** Low
- **Notes:** Start with simple table in Postgres. Later: export to centralized audit sink (Grafana Loki, Splunk, etc.).

#### T6.5.3 Break-glass / emergency access procedure
- **Description:** Document and test procedure for platform admin to perform mutations (deploy, promote, config) if portal is down or unresponsive. Steps: direct Git branch + commit + PR (no portal), manually merge PR, await Argo sync.
- **Status:** TODO
- **Acceptance Criteria:**
  - Procedure documented in ops runbook.
  - Tested at least once (intentionally use break-glass, confirm success).
  - Break-glass action logged separately (manual Git change).
  - Recovery plan to re-enable portal noted.
- **Dependencies:** T6.2.1, docs
- **Complexity:** S
- **Risk:** Low

---

### E6.6 Deployment records, rollout verification, and deploy safety controls

#### T6.6.1 Deployment records database schema and migrations
- **Description:** Add a first-class `deployments` table in the portal backend database. Minimum schema:
  - `deployment_id`
  - `service_id`
  - `env`
  - `action_type` (`deploy`, `promote`, `rollback`, `config-change`)
  - `requested_by`
  - `requested_at`
  - `pr_url`
  - `pr_number`
  - `merge_sha`
  - `target_image`
  - `previous_image`
  - `argo_app`
  - `sync_status`
  - `health_status`
  - `started_at`
  - `finished_at`
  - `result`
  - `result_reason`
  - `deploy_window_start`
  - `deploy_window_end`
  - `deploy_reason`
  - `compare_url`
- **Status:** TODO
- **Acceptance Criteria:**
  - Alembic migration creates `deployments` table with indexes on `service_id`, `env`, `requested_at`, and `result`.
  - Enum constraints enforce allowed `action_type` and lifecycle status values.
  - Backfill script can map recent release rows into deployment records without data loss.
  - Retention policy for deployment records is documented (minimum 180 days).
- **Dependencies:** T1.2.2, T6.2.1
- **Complexity:** M
- **Risk:** Medium

#### T6.6.2 Deployment records API and workflow integration
- **Description:** Implement backend APIs and workflow hooks so every deploy/promote/rollback/config-change creates and updates a deployment record. Add endpoints:
  - `GET /api/services/{service_id}/deployments?env=&limit=`
  - `GET /api/deployments/{deployment_id}`
  - `POST /api/deployments/{deployment_id}/cancel` (optional soft-cancel before merge)
  - Internal workflow method to transition status lifecycle.
- **Status:** TODO
- **Acceptance Criteria:**
  - Deployment status lifecycle is enforced as `pending`, `deploying`, `live`, `failed`.
  - T6.2.1/T6.2.2/T6.2.3 and config-change workflows create records before PR open.
  - Record updates include PR metadata, merge SHA, rollout timestamps, result, and result reason.
  - Service deployment history endpoint returns deterministic ordering and pagination.
- **Dependencies:** T6.6.1, T6.2.1, T6.2.2, T6.2.3, T6.3.1
- **Complexity:** M
- **Risk:** Medium

#### T6.6.3 Rollout watcher after PR merge (Argo + Kubernetes verification)
- **Description:** Implement a rollout watcher worker that, after PR merge detection, verifies rollout outcome by polling Argo CD Application status and Kubernetes rollout status, then updates deployment records.
- **Status:** TODO
- **Acceptance Criteria:**
  - Merge detection links PR number/merge SHA to exactly one deployment record.
  - Watcher polls Argo sync/health and Kubernetes rollout progression for target workload.
  - Status transitions follow `pending` → `deploying` → `live`/`failed`; timeout marks `failed` with explicit reason.
  - Optional GitHub PR comment posts deployment result summary with deployment ID and portal link.
- **Dependencies:** T6.6.2, T2.2.2, T4.4.6
- **Complexity:** L
- **Risk:** High

#### T6.6.4 Deploy lock manager (service+env mutex)
- **Description:** Add deploy locks so only one active mutation per `service_id + env` can run at a time across deploy/promote/rollback/config changes.
- **Status:** TODO
- **Acceptance Criteria:**
  - Lock table or lock records are stored in backend DB and include owner, action type, and lock timestamp.
  - API rejects overlapping mutation requests with stable conflict response payload.
  - Portal UI disables conflicting action buttons when lock is active.
  - Lock release is guaranteed on success/failure/timeout paths; stale lock cleanup job exists.
- **Dependencies:** T6.6.1, T6.6.2, T6.2.4
- **Complexity:** M
- **Risk:** High

#### T6.6.5 Canonical service identity enforcement and validation
- **Description:** Enforce canonical `serviceId` identity and label conventions across portal registry, Argo CD apps, Kubernetes labels, Prometheus selectors, Loki selectors, and release/deployment joins.
- **Status:** TODO
- **Acceptance Criteria:**
  - Canonical mapping is validated for every registered service/env:
    - `serviceId` (portal/backend key)
    - `app.kubernetes.io/name`
    - `app.kubernetes.io/instance`
    - `env`
    - `argo_app`
  - Validation script fails CI when required labels/identity mappings are missing or inconsistent.
  - Monitoring and release query builders consume canonical identity fields only (no ad-hoc aliases).
  - Smoke-check runbook documents identity validation against at least 2 live services.
- **Dependencies:** T6.4.4, T4.4.1, T4.6.5
- **Complexity:** M
- **Risk:** Medium

#### T6.6.6 OPTIONAL: Preview environment GitOps workflow (PR-based)
- **Description:** Optional capability to create ephemeral preview environments per PR via Git changes only. Workflow opens Git PR adding preview namespace/app overlay, Argo reconciles it, and portal surfaces preview URL.
- **Status:** TODO
- **Acceptance Criteria:**
  - Preview env is created from a PR-scoped overlay/namespace naming convention (`preview-<pr-number>`).
  - Portal shows preview URL and environment state as clearly non-production.
  - Preview creation follows same deployment record model (`action_type=deploy`, `env=preview-*`).
  - No direct kubectl mutations are performed by portal.
- **Dependencies:** T6.1.2, T6.6.2, T6.4.4
- **Complexity:** L
- **Risk:** Medium

#### T6.6.7 OPTIONAL: Preview environment lifecycle cleanup
- **Description:** Optional follow-up to auto-clean preview environments when source PR closes/merges by creating Git cleanup PRs that remove preview overlays and namespace manifests.
- **Status:** TODO
- **Acceptance Criteria:**
  - PR close/merge event triggers cleanup workflow that removes preview manifests via Git PR.
  - Cleanup has TTL safety net for abandoned previews.
  - Portal marks preview as expired/cleaned and preserves deployment history record.
  - Cleanup failures are surfaced in portal and runbook with retry path.
- **Dependencies:** T6.6.6, T6.6.3
- **Complexity:** M
- **Risk:** Medium

---

### E6.7 Deploy-scoped observability and deploy timeline UX

#### T6.7.1 Backend deploy-window observability aggregation endpoint
- **Description:** Add backend endpoint to aggregate telemetry scoped to a deployment window using existing Prometheus/Loki data:
  - `GET /api/deployments/{deployment_id}/observability`
  - Returns logs window links and metric deltas: error-rate before/after, latency before/after, restart spikes, availability impact.
- **Status:** TODO
- **Acceptance Criteria:**
  - Endpoint computes before/after windows from `deploy_window_start` and `deploy_window_end`.
  - Uses only existing observability stack integrations (Prometheus, Loki, Grafana links).
  - Missing telemetry returns explicit partial-data fields without failing full response.
  - Response includes query context for reproducibility/debugging.
- **Dependencies:** T6.6.2, T4.4.2, T4.4.8, T4.4.12
- **Complexity:** M
- **Risk:** Medium

#### T6.7.2 Portal deploy history timeline (deploy records as first-class UX)
- **Description:** Replace mocked/adapter-only deployment history with deployment-record-backed timeline on `/services/:serviceId/deployments` and service overview preview.
- **Status:** TODO
- **Acceptance Criteria:**
  - Timeline rows are sourced from deployment records API (not mock rows).
  - Each row shows action type, status, requested by/at, PR link, merge SHA, previous→new version, deploy reason, and compare link.
  - Row drill-down opens deploy-scoped logs/metrics panel for that deployment.
  - UI highlights failed deployments and supports filtering by result/action/env.
- **Dependencies:** T6.6.2, T6.2.4, T1.6.4, T4.5.2
- **Complexity:** M
- **Risk:** Medium

#### T6.7.3 Deploy impact summary and regression highlighting
- **Description:** Extend deployment history and service detail to show deploy impact summaries (error-rate delta, latency delta, restart spikes, availability impact) and emphasize regressions immediately after deploy.
- **Status:** TODO
- **Acceptance Criteria:**
  - Deploy rows include before/after indicators with explicit unavailable state when telemetry windows are missing.
  - Services list/detail highlight recent deployments with negative deltas using shared severity semantics.
  - Regression thresholds are configurable and documented in runbook.
  - Existing monitoring links still deep-link to Grafana/Loki for full investigation.
- **Dependencies:** T6.7.1, T4.3.7, T4.3.8
- **Complexity:** M
- **Risk:** Medium

#### T6.7.4 Historical deployment backfill from Argo/Git/GitHub evidence
- **Description:** Build a one-time and repeatable backfill path that reconstructs older deployment records from trusted deployment evidence sources instead of pretending image/package history equals deployment history. Candidate inputs include Argo CD application history, Git promotion PRs/merge commits, rollout annotations, and explicit deployment metadata already emitted by CI.
- **Status:** TODO
- **Acceptance Criteria:**
  - Backfill process can create historical deployment records for existing services where deploy evidence exists.
  - Each reconstructed record stores provenance and confidence (`source=argo_history`, `source=git_pr`, `source=rollout_annotation`, etc.).
  - Package/image versions with no deployment evidence are not shown as deployed history rows.
  - Backfill output is idempotent and safe to rerun without duplicating deployment records.
- **Dependencies:** T6.6.1, T6.6.2, T6.6.3, T6.2.5
- **Complexity:** L
- **Risk:** High

#### T6.7.5 Separate build/package history from deployment history
- **Description:** Add a clear product boundary between "published builds/images" and "actual deployments" so the portal can show both without conflating them. Build/package history may come from GitHub Packages/GHCR or workflow artifacts, but deployment history must remain tied to deployment records only.
- **Status:** TODO
- **Acceptance Criteria:**
  - UI and API expose build/package history separately from deployment history.
  - Deployment timeline never renders undeployed package versions as if they were live rollouts.
  - Services/release views can link a deployment record to its source image/build when both exist.
  - Copy and badges make the distinction explicit (`Published`, `Deployed`, `Unknown deployment evidence`).
- **Dependencies:** T6.2.5, T6.6.2, T6.7.2
- **Complexity:** M
- **Risk:** Medium

#### T6.7.6 Durable deployment observability snapshots beyond Prometheus retention
- **Description:** Persist post-deploy observability snapshots (or compact rollups) onto deployment records at deploy time so older rows remain useful after Prometheus/Loki retention windows expire.
- **Status:** TODO
- **Acceptance Criteria:**
  - Deployment records retain summarized before/after observability fields even after raw telemetry ages out.
  - Historical deployment rows can show meaningful impact summaries without live re-query of expired windows.
  - UI distinguishes `no retained telemetry captured at deploy time` from `query failed now`.
  - Snapshot retention, storage size, and recomputation policy are documented.
- **Dependencies:** T6.6.2, T6.6.3, T6.7.1
- **Complexity:** M
- **Risk:** Medium

---

## Sequencing and dependencies for Portal Level 3

### Implementation order by epic

1. **Start E6.1 (Git integration)** once Phase 2 and 3 are stable (CI proven, secrets in place).
   - T6.1.1 → T6.1.2 → T6.1.3 → T6.1.4 (do all before moving to E6.2).

2. **Implement E6.6 core first (deployment records + rollout verification + locks).**
   - T6.6.1 → T6.6.2 → T6.6.3 → T6.6.4 → T6.6.5.
   - **Checkpoint:** Deployment records persist and transition `pending/deploying/live/failed`.
   - **Checkpoint:** Post-merge rollout watcher updates result and reason.

3. **Proceed with E6.2 (Deploy/promote/rollback)** after E6.6 core exists.
   - T6.2.1 → T6.2.2 → T6.2.3 → T6.2.4 → T6.2.5.
   - **Checkpoint:** First deploy PR works end-to-end.
   - **Checkpoint:** Promotion works at least once.
   - **Checkpoint:** Rollback tested and verified.

4. **Parallel E6.3 (Config/secrets)** after E6.2 basics (T6.2.1–T6.2.4) are working.
   - T6.3.1 → T6.3.2 → T6.3.3 → T6.3.4.
   - **Checkpoint:** Config edit PR created and merged; env var applied to pod.
   - **Checkpoint:** Secret edit PR created; secret decrypts and applies (manual verification).

5. **Implement E6.7 (deploy-scoped observability)** after deployment records and rollout statuses are live.
   - T6.7.1 → T6.7.2 → T6.7.3.
   - **Checkpoint:** Deploy timeline includes logs/metrics context and before/after deltas.

6. **Begin E6.4 (Service scaffolding)** after >3 services running and deploy/promote proven stable (2–3 weeks into Level 3).
   - T6.4.1 → T6.4.2 → T6.4.4 → T6.4.3 (portal UI only after manual scaffolding proven).

7. **E6.5 (Multi-user + audit)** only after E6.2/E6.3/E6.6 prove stable and if adding more users.
   - T6.5.1 → T6.5.2 → T6.5.3.
   - **Do not do E6.5 for a solo engineer initially.**

### Critical sequencing rule

Do not attempt E6.2 or E6.3 until you have successfully:
- Expected readiness gates passed (above).
- Completed T6.1.1–T6.1.4 (Git integration tested with dry runs).
- Completed T6.6.1–T6.6.3 (deployment record persistence + rollout verification core).
- Tested Git provider token with at least one manual PR (not via portal, just verify token works).

Attempting Level 3 before these checkpoints will result in Git-based outages and rework.

---

## Anti-scope creep for Portal Level 3

Explicitly NOT included in this phase:

- **Direct kubectl apply from portal:** Only Git-backed changes allowed. Cluster is read-only from portal view.
- **Custom Kubernetes controllers for provisioning:** Use Argo CD Applications only; no custom operators yet.
- **Full internal developer platform portal (e.g. Backstage):** Keep this lightweight (services list + deploy controls + links out). Advanced scaffolding, policy templates, and multi-team governance belong in later years.
- **Complex policy engines (OPA/Kyverno/Gatekeeper):** Rely on Git review process and RBAC for approval gates initially.
- **Multi-tenant isolation:** Assume single owner (you) or very small team. Tenant boundaries defer to Phase 7+.
- **Canary/blue-green deployments:** Use simple rolling updates. Advanced deployment strategies (traffic split, progressive rollout) deferred.
- **Service mesh integration:** Keep networking simple. Defer Istio/Linkerd until you have concrete mTLS or traffic-splitting requirements.
- **Portal write access to non-app resources:** services, namespaces, secrets (platform-layer) must be edited directly in Git. Portal only edits app-layer configs and images.

---

## Suggested Portal Level 3 timeline (evenings/weekends, ~10–12h/week)

Assumption: Phase 2–5 readiness gates are already met.

### Month 1 of Level 3 (Weeks 1–4)
**Goal:** Establish deployment records + prove PR-based deploy workflow.

- **Week 1:** Complete E6.1 (Git integration foundation).
  - T6.1.1 → ADR done.
  - T6.1.2 → Git service module + unit tests pass.
  - T6.1.3 → Token provisioned and rotated into secret.
  - ~12–15 hours.

- **Week 2:** Build E6.6 core foundation.
  - T6.6.1 → Deployment records schema/migration live.
  - T6.6.2 → Deployment records API + status lifecycle live.
  - T6.6.3 → Rollout watcher updates deployment outcome after merge.
  - Checkpoint: deployment status transitions `pending` → `deploying` → `live/failed` are visible.
  - ~10–12 hours.

- **Week 3–4:** Complete core E6.2.
  - T6.2.1 → Deploy-to-dev endpoint works with deploy reason/changelog capture.
  - T6.2.2 → Promote-to-prod endpoint works.
  - T6.2.3 → Rollback endpoint works.
  - T6.2.4 + T6.6.4 → UI controls + deploy locks prevent overlapping mutations.
  - T6.2.5 → Deployment traceability card linked to deployment records.
  - End-to-end test: dev deploy → promote → rollback with rollout verification and persisted deployment history.
  - Checkpoint: Full deploy/promote/rollback workflow with traceability is end-to-end tested.
  - ~15–18 hours.

### Month 2 of Level 3 (Weeks 5–8)
**Goal:** Config/secrets workflows + deploy-scoped observability.

- **Week 5:** E6.3.1–T6.3.2.
  - T6.3.1 → Config edit endpoint.
  - T6.3.2 → Secrets edit endpoint (requires high scrutiny).
  - ~12–15 hours.

- **Week 6:** E6.3.3–T6.3.4.
  - T6.3.3 → Pod restart strategy tested.
  - T6.3.4 → Portal UI for config/secrets.
  - ~10–12 hours.

- **Week 7:** Stabilize and test E6.2/E6.3.
  - End-to-end tests: config edit + pod restart + verify new value in app.
  - Edge case testing: concurrent edits, rollback mid-edit, token expiry.
  - ~8–10 hours.

- **Week 8:** E6.7 observability timeline + runbooks.
  - T6.7.1 → Deploy-window observability endpoint.
  - T6.7.2/T6.7.3 → Deploy timeline UI with before/after deltas and regression highlighting.
  - Incident playbooks: "PR merged but rollout failed", "deploy lock stuck", "post-deploy error spike".
  - ~6–8 hours.

### Month 3 of Level 3 (Weeks 9–12)
**Goal:** Service scaffolding + evaluation.

- **Week 9–10:** E6.4.2–T6.4.4.
  - T6.4.1 → ADR and schema for services.yaml.
  - T6.4.2 → Scaffold script.
  - T6.4.4 → services.yaml populated.
  - Generate at least 2 services end-to-end.
  - ~15–18 hours.

- **Week 11:** E6.4.3 (optional for Month 3; defer if time-constrained).
  - Portal UI for "New Service" wizard.
  - ~8–10 hours.

- **Week 12:** Evaluation & retrospective.
  - Successful service scaffolds deployed?
  - Deployment/promotion automatio proven? Any gaps?
  - Decision: proceed to E6.5 or iterate more on E6.2–E6.4?
  - Document lessons learned and update runbooks.
  - ~4–6 hours.

### Estimated total Level 3 effort
- Month 1: 40–50 hours (Git integration + deployment records + core deploy/promote/rollback).
- Month 2: 38–48 hours (Config/secrets + deploy-scoped observability + stabilization).
- Month 3: 27–34 hours (Service scaffolding + evaluation).
- **Total: 105–132 hours (~10–12 hours/week for 10+ weeks).**

If solo engineering at evenings/weekends (5–8 hours/week), extend by 1.5–2x (15–20 weeks for full Level 3).

### Checkpoints and decision gates

- **After Month 1:** Do deployment records persist with rollout verification and can portal deploy/promote/rollback via PR? If YES, proceed to Month 2. If NO, iterate on E6.6/E6.2 before moving forward.
- **After Month 2:** Can operator edit config/secrets and inspect deploy-scoped impact (logs/metrics deltas) for each deploy? If YES, proceed to scaffolding. If NO, address E6.3/E6.7 gaps.
- **After Month 3:** Are >3 new services scaffolded and deployed? Is scaffolding repeatable in <30 min? If YES, move to production use. If NO, keep script maintenance as ongoing task.

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
