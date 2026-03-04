# Platform Health Dashboard Runbook

This runbook covers T4.1.2 for a single dashboard view of app availability, deployment status, and prototype error thresholds.

## 1. Source of truth

Dashboard and alerts are defined in:

1. `workloads/environments/dev/workloads/monitoring-app.yaml`

Implemented as:

1. Grafana dashboard `Homelab Platform Health` via `grafana.dashboards.default`.
2. Prometheus alert rules via `additionalPrometheusRulesMap.homelab-platform-health`.

## 2. Dashboard coverage

Dashboard panels included:

1. API availability ratio (`homelab-api` available/desired replicas).
2. Frontend availability ratio (`homelab-web` available/desired replicas).
3. DB ready replicas (`homelab-api-postgres` StatefulSet).
4. API restart count over 15 minutes.
5. Deployment ready vs desired replicas for API and web.
6. Prototype API 5xx rate panel from Traefik metrics.

## 3. Alert threshold prototypes

Prototype alerts configured:

1. `HomelabApiUnavailable` (`< 1` available replicas for 5m).
2. `HomelabWebUnavailable` (`< 1` available replicas for 5m).
3. `HomelabPostgresUnavailable` (`< 1` ready replicas for 5m).
4. `HomelabApiRestartSpike` (restart increase `> 2` in 15m for 10m).
5. `HomelabApi5xxRateHighPrototype` (5xx RPS `> 0.05` for 10m, Traefik metric dependent).

## 4. Validation

Apply monitoring app:

```bash
kubectl apply -k workloads/environments/dev/workloads
```

Check Argo status:

```bash
argocd app get monitoring-dev --grpc-web
```

Open Grafana:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Then verify dashboard exists:

1. Dashboards -> search for `Homelab Platform Health`.
2. Confirm panels render for:
   - API availability
   - frontend availability
   - DB ready replicas
   - deployment ready vs desired

Verify alert rules loaded:

```bash
kubectl -n monitoring get prometheusrules | rg homelab-platform-health
kubectl -n monitoring get prometheusrule -o yaml | rg -n "HomelabApiUnavailable|HomelabWebUnavailable|HomelabPostgresUnavailable|HomelabApiRestartSpike|HomelabApi5xxRateHighPrototype"
```

## 5. Notes

1. API 5xx prototype alert depends on Traefik metrics (`traefik_service_requests_total`) being present.
2. If Traefik metrics are not enabled, this panel/alert remains inactive and should be treated as a prototype placeholder.
3. If all `kube_*` panels show no data, validate `up{job="kube-state-metrics"}` first; if empty, confirm Argo CD `argocd-cm` sets `application.instanceLabelKey: argocd.argoproj.io/instance` so ServiceMonitor label selectors are not broken.
