# Service Identity Contract

Defines the canonical service identity used by portal monitoring adapters and backend monitoring APIs.

## Shape

```ts
interface ServiceIdentity {
  serviceId: string
  serviceName: string
  namespace: string
  env: string
  appLabel: string
  argoAppName?: string
}
```

## Field semantics

- `serviceId`: stable portal service key used in route/API paths (for example `homelab-api`).
- `serviceName`: display name for UI and operator workflows.
- `namespace`: Kubernetes namespace used for logs/metrics scoping.
- `env`: environment discriminator (`dev`, `prod`, etc).
- `appLabel`: canonical app selector for Loki/Prometheus joins.
- `argoAppName`: optional explicit Argo CD app name; defaults to `<serviceId>-<env>` when omitted.

## Usage rules

1. Monitoring adapters accept `ServiceIdentity` as input (or derive it from `serviceId` when only ID is available).
2. `services` adapter is the source of truth for deriving identity for service rows and detail routes.
3. When backend payload omits identity fields, frontend applies safe defaults:
   - `namespace=default`
   - `env=dev`
   - `appLabel=serviceId`
   - `argoAppName=serviceId-env`

## Examples

### Example A: homelab-api in dev

```json
{
  "serviceId": "homelab-api",
  "serviceName": "homelab-api",
  "namespace": "default",
  "env": "dev",
  "appLabel": "homelab-api",
  "argoAppName": "homelab-api-dev"
}
```

### Example B: homelab-web in prod

```json
{
  "serviceId": "homelab-web",
  "serviceName": "homelab-web",
  "namespace": "default",
  "env": "prod",
  "appLabel": "homelab-web",
  "argoAppName": "homelab-web-prod"
}
```

## Implemented references

- `apps/portal/frontend/src/lib/service-identity.ts`
- `apps/portal/frontend/src/lib/adapters/services.ts`
- `apps/portal/frontend/src/lib/adapters/deployments.ts`
- `apps/portal/frontend/src/lib/adapters/service-metrics.ts`
- `apps/portal/frontend/src/lib/adapters/service-health-timeline.ts`
- `apps/portal/frontend/src/lib/adapters/platform-health.ts`
