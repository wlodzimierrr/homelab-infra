# Service Adoption & Namespace Migration

Migrate standalone services into a shared project namespace. This is a two-phase process:
Phase 1 (soft-link) is metadata-only and safe to reverse. Phase 2 (namespace consolidation) moves Kubernetes resources and requires careful validation.

## Prerequisites

- Portal backend running with admin access
- `workloads` repo accessible via `GITOPS_WORKLOADS_REPO_URL`
- ArgoCD syncing the workloads repo

## Phase 1: Soft-Link (Metadata Only)

### When to use

Two standalone services should be grouped under one project for catalog views, but you are not ready (or don't need) to consolidate namespaces.

### How to apply

Use the portal UI "Adopt into Project" button on the service detail page, or call the API directly:

```bash
curl -X POST \
  -H 'Authorization: Bearer dev-static-token' \
  -H 'Content-Type: application/json' \
  -d '{"projectId": "my-project"}' \
  http://api.dev.homelab.local/services/my-backend/adopt
```

This creates a PR that adds `project_id: my-project` to the service's entry in `services.yaml`.

### What changes

- `services.yaml`: `project_id` field added to the service entry
- After PR merge + catalog sync: `service_registry.project_id` column is stamped
- Portal Projects page shows the service under the parent project
- Portal Service detail page shows sibling services

### Rollback

1. Remove the `project_id` line from the service's entry in `services.yaml`
2. Commit, push, let ArgoCD sync
3. Re-run catalog sync: `POST /service-registry/sync?source=gitops_apps`

## Phase 2: Namespace Consolidation

### When to use

After soft-linking, you want to move a service into the project's shared namespace so services can communicate via cluster-local DNS without cross-namespace network policies.

### Pre-flight validation

Always validate before consolidating:

```bash
curl -X POST \
  -H 'Authorization: Bearer dev-static-token' \
  -H 'Content-Type: application/json' \
  -d '{"serviceId": "my-backend", "targetProjectId": "my-project", "targetNamespace": "my-project"}' \
  http://api.dev.homelab.local/migration/validate
```

The response lists conflicts by severity:

- **error**: Blocks migration (e.g., PVC cannot be moved, ingress host collision)
- **warning**: Requires attention but migration can proceed (e.g., secret must be recreated, DNS references need updating)

### Conflict types

| Conflict | Severity | Description |
|----------|----------|-------------|
| `ingress_host_conflict` | error | Two services claim the same host+path in the target namespace |
| `pvc_scope` | error | PersistentVolumeClaims are namespace-scoped and cannot be moved |
| `argo_app_conflict` | error | Source service has its own ArgoCD Application that conflicts with the target project's app |
| `secret_scope` | warning | Secrets referenced by the service don't exist in the target namespace |
| `service_discovery` | warning | Environment variables reference the old namespace DNS name |

### Expected downtime

Service is unavailable from the moment the ArgoCD sync deletes resources in the old namespace until they are recreated in the new namespace. Typically 30-60 seconds.

### Execution

```bash
curl -X POST \
  -H 'Authorization: Bearer dev-static-token' \
  -H 'Content-Type: application/json' \
  -d '{"serviceId": "my-backend", "targetProjectId": "my-project", "targetNamespace": "my-project", "acknowledgeConflicts": true}' \
  http://api.dev.homelab.local/migration/consolidate
```

This creates a PR that:
- Rewrites `metadata.namespace` in all base manifests for the service
- Moves manifest files into the target project's `apps/{project}/base/` directory
- Updates `services.yaml` with the new namespace and project_id
- Adds the service's resources to the target project's `kustomization.yaml`
- Removes the standalone ArgoCD Application (the project's app covers the namespace)

### Rollback

1. Revert the PR (or create a revert PR)
2. ArgoCD syncs resources back to the original namespace
3. Secrets and ConfigMaps may need to be recreated in the original namespace

### Post-migration validation

1. Check ArgoCD sync status for the target project
2. Verify health endpoints respond:
   ```bash
   curl -s http://my-backend.dev.homelab.local/health
   ```
3. Verify service appears under the project in the portal:
   ```bash
   curl -sS -H 'Authorization: Bearer dev-static-token' \
     http://api.dev.homelab.local/catalog/reconciliation | jq '.rows[] | select(.projectId == "my-project")'
   ```
4. Clean up the old empty namespace if no other resources remain
