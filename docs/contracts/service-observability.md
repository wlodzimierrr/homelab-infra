# Service Observability Contract

Defines the minimum observability baseline every service must declare and the authoritative HTTP metrics source the portal should treat as canonical.

## Goals

- Keep service observability universal as more workloads are added.
- Make HTTP metrics expectations explicit in Git instead of inferred from implementation details.
- Let portal diagnostics explain why a service has rich latency/error data, ingress-only signals, or intentionally no HTTP metrics.

## Baseline required for every service

Every service must expose or inherit:

- canonical service identity:
  - `service_id`
  - `namespace`
  - `app_label`
  - `argo_app`
- logs visibility through the platform log path
- uptime and restart visibility through Kubernetes or platform metrics
- deploy-window visibility through deployment records
- one declared HTTP metrics mode in `services.yaml`

## HTTP metrics modes

### `app-native`

Use for services that expose their own HTTP metrics and are scraped directly.

Required baseline:

- app-level Prometheus metrics for request duration and request counts
- scrape configuration such as a `ServiceMonitor`
- canonical `namespace` and `app_label` selectors

Portal behavior:

- app metrics are authoritative for latency and error rate
- ingress-derived metrics are fallback only

### `ingress-derived`

Use for routed HTTP services that do not expose their own application metrics.

Required baseline:

- service is reachable through the ingress layer
- ingress metrics are available and label-mappable to the service identity

Portal behavior:

- ingress metrics are authoritative for latency and error rate
- app-level HTTP metrics are not required

### `no-http`

Use for support services, background jobs, or components where HTTP latency and error rate are not primary signals.

Required baseline:

- logs
- uptime or readiness where applicable
- restart visibility

Portal behavior:

- latency and error-rate cards should explain that HTTP metrics are intentionally not expected
- missing HTTP telemetry is not treated as drift for this mode

## Git declaration

Each top-level service entry in `workloads/services.yaml` must declare:

```yaml
services:
  - service_id: homelab-api
    name: Homelab API
    observability:
      mode: app-native
```

Allowed values:

- `app-native`
- `ingress-derived`
- `no-http`

This declaration is part of service scaffolding and should be added for every new service at creation time.

## Runtime expectations

The portal should surface the declared mode in:

- project metadata
- service details
- service registry diagnostics

Diagnostics should fail or warn when:

- a service catalog entry omits the mode
- the mode is invalid
- future enforcement layers detect that the declared mode does not match the live scrape path

## Current assignments

- `homelab-api`: `app-native`
- `homelab-web`: `ingress-derived`

Future support or background services can use `no-http` when HTTP latency and error rate are not meaningful signals.
