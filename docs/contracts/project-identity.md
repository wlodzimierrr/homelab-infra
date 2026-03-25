# Project Identity Contract

Defines the project-level grouping contract that owns namespaces and contains one or more services.

## Shape

```ts
interface ProjectIdentity {
  projectId: string
  projectName: string
  env: string
  namespace: string
  serviceIds: string[]
  publicHost?: string
  owner: string
  repoUrl: string
}
```

## Field semantics

- `projectId`: stable project key derived from `normalize_service_id()`. Unique per env.
- `projectName`: display name for UI and operator workflows.
- `env`: environment discriminator (`dev`, `prod`, etc).
- `namespace`: Kubernetes namespace owned by the project. All services within a project share this namespace in the given env.
- `serviceIds`: list of `service_id` values that belong to this project. For single-service projects this is `[projectId]`.
- `publicHost`: optional public DNS hostname for the project's primary ingress.
- `owner`: owner identifier (team or individual).
- `repoUrl`: source repository URL.

## Relationship to ServiceIdentity

- A project contains one or more services.
- Each service belongs to exactly one project, declared via `project_id` in `services.yaml` or inferred as `project_id = service_id` when omitted.
- Namespace ownership is at the project level: the project decides the namespace, and all child services inherit it for the same env.
- Services within a project may have distinct `app_label`, `argo_app`, and `observability.mode` values.

## Git declaration

Each service entry in `workloads/services.yaml` may declare an optional `project_id`:

```yaml
services:
  # Single-service project (project_id defaults to service_id)
  - service_id: homelab-api
    name: Homelab API
    owner: wlodzimierrr
    observability:
      mode: app-native
    envs:
      - name: dev
        namespace: homelab-api
        argo_app: homelab-api-dev

  # Support service belonging to another project
  - service_id: oauth2-proxy
    project_id: homelab-web
    name: OAuth2 Proxy
    observability:
      mode: no-http
    envs:
      - name: dev
        namespace: homelab-web
        app_label: oauth2-proxy
        argo_app: homelab-web-dev
        workload_ref: apps/homelab-web/envs/dev/oauth2-proxy.yaml
```

When `project_id` is absent, it defaults to `service_id`. This preserves backward compatibility: all existing services are implicitly single-service projects.

## Database representation

### project_registry

Primary key: `(project_id, env)`. Stores project-level metadata: namespace, owner, repo_url, observability_mode, public_host.

### service_registry

Primary key: `(service_id, env)`. Optional `project_id` column links to the parent project. When NULL, the service is its own project (`project_id = service_id`).

## Catalog reconciliation

The catalog join between project_registry and service_registry uses two strategies:

1. **Explicit join**: when `service_registry.project_id` is set, match directly to `project_registry.project_id + env`.
2. **Heuristic join**: when `project_id` is NULL, match on `(env, namespace, app_label)` as before.

One-to-many joins (one project matching multiple services) are first-class, not diagnostic anomalies. The `CatalogJoinRow` result includes `serviceIds: string[]` and `serviceCount: number`.

## Backward compatibility

- Existing services without `project_id` in `services.yaml` are treated as single-service projects where `projectId = serviceId`.
- No data migration is needed for existing entries.
- API endpoints continue to accept `service_id` as a lookup key. The `project_id = service_id` fallback is preserved in monitoring context resolution.

## Implemented references

- `docs/contracts/service-identity.md`
- `docs/contracts/service-observability.md`
- `apps/portal/backend/app/gitops_project_sync.py`
- `apps/portal/backend/app/catalog_reconciliation.py`
- `apps/portal/backend/app/service_registry_sync.py`
- `workloads/services.yaml`
