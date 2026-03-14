# 0008: Service metadata storage and registration model

- Status: Accepted
- Date: 2026-03-13
- Owners: @wlodzimierrr

## Context

The homelab portal needs a source of truth for service metadata: the structured set of facts about each service
that is not derivable from a running deployment alone. This includes:

- Human-readable name and description
- Owner
- Source code repository URL
- Runbook URL
- Per-environment mapping (namespace, Argo CD Application name, app label)
- Observability mode (`app-native`, `ingress-derived`, `no-http`)
- Future fields: tags, on-call contact, SLO tier

Three storage models were considered:

**Option A — Git-owned manifest (`services.yaml` or `services/` directory in the workloads repo)**

Service metadata lives in a YAML file committed to the `homelab-workloads` repository, colocated with the
Kubernetes manifests it describes. The portal backend reads this file on startup or on a sync interval and
caches it in the PostgreSQL service registry table.

**Option B — Backend database only**

Service records are created and maintained exclusively through the portal API. The database is the sole
source of truth; no Git file is involved.

**Option C — Argo CD Application labels/annotations only**

Service metadata is stored as labels or annotations on Argo CD Application resources in the cluster.
The portal queries the Kubernetes API to resolve service identity.

## Decision

**Option A accepted.** Service metadata is stored in `workloads/services.yaml`, a Git-owned YAML manifest
in the workloads repository. The portal backend syncs this file into its PostgreSQL service registry table.

The manifest schema for each entry:

```yaml
services:
  - service_id: <kebab-case unique identifier>
    name: <human readable display name>
    owner: <GitHub username or team>
    repo_url: <URL to the service source code>
    runbook_url: <URL to the operational runbook>
    description: <one-line description>
    observability:
      mode: app-native | ingress-derived | no-http
    envs:
      - name: dev | prod
        namespace: <Kubernetes namespace>
        argo_app: <Argo CD Application name>
        app_label: <optional override; defaults to service_id>
        workload_ref: <optional path to specific workload manifest>
```

This schema is already in production use in `workloads/services.yaml`.

## Alternatives considered

**Option A — Git-owned manifest (accepted)**
- Pros: auditable history; PR-based registration; no DB schema migrations for metadata changes;
  colocated with the manifests it describes; zero additional infrastructure; consistent with the
  existing GitOps model.
- Cons: changes require a Git commit + sync cycle (seconds to minutes latency, not real-time);
  YAML is less queryable than a relational table for complex filtering.
- Accepted because the homelab scale (tens of services, not thousands) makes latency and query
  complexity non-issues.

**Option B — Backend database only**
- Pros: real-time updates; full SQL queryability; portal UI can self-register services without
  repository access.
- Cons: database is the only source of truth — no audit trail without additional change-data-capture;
  metadata diverges from the GitOps manifests; requires schema migrations to add new fields;
  data is lost on DB wipe without a separate backup strategy.
- Rejected because it violates the GitOps principle that Git is the system of record, and adds
  operational burden (backup, migration) with no benefit at homelab scale.

**Option C — Argo CD Application labels/annotations only**
- Pros: zero additional files; metadata co-located with the live deployment.
- Cons: Argo CD Application annotations are not designed as a metadata store; limited schema;
  no structured typing or validation; querying requires Kubernetes API access; adding non-Argo fields
  (owner, runbook URL) is non-standard and brittle.
- Rejected because it conflates deployment configuration with service catalogue metadata and is
  not extensible to the fields required (observability mode, owner, runbook URL).

## Consequences

- Positive: all service metadata is version-controlled and auditable.
- Positive: onboarding a new service is a single Git PR to `workloads/services.yaml` — reviewable
  and mergeable like any other infrastructure change.
- Positive: portal reads from the database cache (fast), which is periodically reconciled from the
  Git file (catalog-sync CronJob).
- Positive: disaster recovery is trivial — re-sync from Git at any time to rebuild the registry.
- Tradeoff: metadata changes are not instantaneous; they depend on a sync cycle (catalog-sync CronJob
  runs on a configured interval, default 10 minutes).
- Tradeoff: `services.yaml` must be kept in sync with actual deployed namespaces/Argo Apps;
  stale entries are possible if a service is deleted without updating the manifest.
- Migration path to Option B: if scale demands it (hundreds of services, complex ownership queries,
  multi-team self-service registration without Git access), the sync model can be inverted — the DB
  becomes the write target and `services.yaml` becomes an export/snapshot. The portal API already
  has the service registry table; only the write path and ownership semantics would change.
- Follow-up: enforce `services.yaml` schema validation in CI (JSON Schema or Pydantic model)
  to catch malformed entries before they reach the portal.
