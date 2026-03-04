# Loki + Promtail Runbook (Homelab)

This runbook covers deployment and validation of centralized logging for T4.2.1.

## 1. Scope and paths

Argo CD applications:

1. `workloads/environments/dev/workloads/loki-app.yaml`
2. `workloads/environments/dev/workloads/promtail-app.yaml`
3. Grafana datasource wiring in `workloads/environments/dev/workloads/monitoring-app.yaml`

Chart sources:

1. Helm repo: `https://grafana.github.io/helm-charts`
2. Charts: `loki`, `promtail`

## 2. Homelab logging profile

1. Loki deployment mode: `SingleBinary`
2. Loki storage: filesystem + PVC (`10Gi`)
3. Retention: `168h` (7 days)
4. Promtail shipping target: `http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push`
5. Query labels: `namespace`, `container`, `app`

## 3. Deploy and sync

```bash
kubectl apply -f workloads/bootstrap/project-homelab.yaml
kubectl apply -k workloads/environments/dev/workloads
```

Check sync:

```bash
argocd app get loki-dev --grpc-web
argocd app get promtail-dev --grpc-web
```

## 4. Validate ingestion and search labels

Check pods:

```bash
kubectl -n monitoring get pods -l app.kubernetes.io/name=loki
kubectl -n monitoring get pods -l app.kubernetes.io/name=promtail
```

Port-forward Loki and test labels:

```bash
kubectl -n monitoring port-forward svc/loki 3100:3100
curl -s 'http://127.0.0.1:3100/loki/api/v1/labels' | jq
curl -s 'http://127.0.0.1:3100/loki/api/v1/label/namespace/values' | jq
curl -s 'http://127.0.0.1:3100/loki/api/v1/label/container/values' | jq
curl -s 'http://127.0.0.1:3100/loki/api/v1/label/app/values' | jq
```

Expected: non-empty values for namespaces and containers; `app` contains values derived from app labels or pod name fallback.

Grafana validation:

1. Open Grafana Explore.
2. Select datasource `Loki`.
3. Run sample query:
   - `{namespace=~"homelab-api|homelab-web"} |= "error"`

## 5. Validate retention

Read Loki runtime config:

```bash
kubectl -n monitoring exec sts/loki -c loki -- sh -c \
  'wget -qO- http://127.0.0.1:3100/config | grep -n retention_period'
```

Expected: `retention_period: 168h`.

## 6. First-run behavior and troubleshooting

During initial rollout, these transient errors are expected:

1. `connect: connection refused` from Promtail to Loki while Loki is still starting.
2. `timestamp too old` (HTTP 400) when Promtail replays historical node log files older than Loki acceptance window.

How to evaluate health:

1. Ensure Loki and Promtail workloads become Ready:
   - `kubectl -n monitoring rollout status statefulset/loki --timeout=5m`
   - `kubectl -n monitoring rollout status daemonset/promtail --timeout=5m`
2. Confirm labels are present:
   - `namespace`, `container`, `app` visible via Loki label APIs.
3. Confirm recent data ingestion:
   - Query a 15-30 minute window and verify non-empty results for active namespaces.

Action threshold:

1. Ignore first-run `timestamp too old` if current logs are ingesting successfully.
2. Investigate only if connection errors continue after Loki is Ready, or if recent-window queries stay empty for active workloads.

## 7. Rollback

If logging rollout causes pressure:

1. Disable auto-sync for `loki-dev` and `promtail-dev` in Argo CD.
2. Revert the manifest commit.
3. Resync both apps and verify `monitoring` namespace returns to the previous state.
