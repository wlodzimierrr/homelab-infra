# Monitoring Provider Readiness and Connectivity Runbook

This runbook validates `T4.7.5` for Prometheus, Loki, and Alertmanager provider readiness.

## 1. Scope

Provider diagnostics endpoints:

- `GET /api/health?includeProviders=true`
- `GET /api/monitoring/providers/diagnostics`

Monitoring endpoints now expose structured provider metadata:

- `GET /api/services/:serviceId/metrics/summary`
- `GET /api/services/:serviceId/logs/quickview`
- `GET /api/alerts/active`

Provider status shape:

- `{ provider, baseUrl, status, reachable, checkedAt, correlationId?, latencyMs?, httpStatus?, error?, probePath? }`

Expected status values:

- `healthy`: provider probe/query succeeded
- `unreachable`: DNS, TCP, timeout, or other network-path failure
- `auth_error`: provider answered but rejected auth (`401` or `403`)
- `http_error`: provider answered with other non-2xx HTTP status
- `bad_payload`: provider returned an unexpected payload shape/status

## 2. Validation steps

1. Call `GET /api/monitoring/providers/diagnostics` with auth.
2. Verify three providers are reported: `prometheus`, `loki`, `alertmanager`.
3. Confirm `overallStatus=healthy` only when all three providers report `status=healthy`.
4. Call `GET /api/health?includeProviders=true` and confirm the same provider set is exposed for lightweight health checks.
5. Call metrics, logs, and alerts endpoints and verify `providerStatus` is present in successful responses.
6. Force a provider failure and verify:
   - metrics/logs return HTTP `502`
   - error payload includes `message`, `correlationId`, and `providerStatus`
   - alerts return HTTP `200` with `alerts=[]` and degraded `providerStatus`

## 3. Cluster checks

### URL and service discovery

1. Confirm configured backend URLs:
   - `PROMETHEUS_BASE_URL`
   - `LOKI_BASE_URL`
   - `ALERTMANAGER_BASE_URL`
2. Resolve the in-cluster services:

```bash
kubectl -n monitoring get svc prometheus-operated alertmanager-operated
kubectl -n monitoring get svc | rg 'loki|grafana|prometheus|alertmanager'
kubectl -n monitoring get endpoints prometheus-operated alertmanager-operated
```

3. From the backend pod, verify DNS and TCP reachability:

```bash
kubectl -n homelab exec deploy/homelab-api -- getent hosts prometheus.monitoring.svc.cluster.local
kubectl -n homelab exec deploy/homelab-api -- getent hosts loki.monitoring.svc.cluster.local
kubectl -n homelab exec deploy/homelab-api -- getent hosts alertmanager-operated.monitoring.svc.cluster.local
kubectl -n homelab exec deploy/homelab-api -- sh -lc 'nc -vz prometheus.monitoring.svc.cluster.local 9090'
kubectl -n homelab exec deploy/homelab-api -- sh -lc 'nc -vz loki.monitoring.svc.cluster.local 3100'
kubectl -n homelab exec deploy/homelab-api -- sh -lc 'nc -vz alertmanager-operated.monitoring.svc.cluster.local 9093'
```

### Auth and backend identity

1. If diagnostics show `auth_error`, confirm the provider is not expecting extra auth headers or ingress auth for cluster-local access.
2. Probe readiness endpoints from inside the backend pod:

```bash
kubectl -n homelab exec deploy/homelab-api -- sh -lc 'wget -qO- http://prometheus.monitoring.svc.cluster.local:9090/-/healthy'
kubectl -n homelab exec deploy/homelab-api -- sh -lc 'wget -qO- http://loki.monitoring.svc.cluster.local:3100/ready'
kubectl -n homelab exec deploy/homelab-api -- sh -lc 'wget -qO- http://alertmanager-operated.monitoring.svc.cluster.local:9093/-/ready'
```

3. If any probe returns `401` or `403`, inspect provider ingress/service exposure and confirm the backend is using the cluster-internal URL, not an externally authenticated route.

### Network policy and namespace isolation

1. List policies in the backend and monitoring namespaces:

```bash
kubectl -n homelab get networkpolicy
kubectl -n monitoring get networkpolicy
```

2. Describe deny/allow rules that could block `homelab-api -> monitoring` traffic:

```bash
kubectl -n homelab describe networkpolicy
kubectl -n monitoring describe networkpolicy
```

3. If traffic is blocked, add or fix an allow rule for:
   - source namespace/pod labels matching the portal backend
   - destination namespace `monitoring`
   - ports `9090`, `3100`, and `9093`

## 4. Failure interpretation

- `reachable=false` with `status=unreachable`: DNS, service, endpoint, timeout, or network-policy path is broken.
- `reachable=true` with `status=auth_error`: provider route is reachable but auth or routing is wrong.
- `reachable=true` with `status=http_error`: provider answered with a non-auth upstream error.
- `status=bad_payload`: provider answered, but the response shape does not match the backend contract.

Use `correlationId` from the API failure payload to correlate backend logs with the failing request.

## 5. Evidence files

- `apps/portal/backend/app/main.py`
- `apps/portal/backend/app/monitoring_providers.py`
- `apps/portal/backend/tests/test_api.py`
- `apps/portal/backend/tests/test_monitoring_providers.py`
