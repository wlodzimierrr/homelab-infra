# Monitoring Query Templates

This document defines backend query-template knobs used by portal observability endpoints.

## Metrics summary templates

- `OBS_QUERY_METRICS_UPTIME`
  - default:
    - `100 * avg_over_time(up{namespace="{namespace}", app="{app_label}"}[{selected_range}])`
- `OBS_QUERY_METRICS_P95_LATENCY`
  - default:
    - `1000 * histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="{namespace}", app="{app_label}"}[5m])))`
- `OBS_QUERY_METRICS_ERROR_RATE`
  - default:
    - `100 * (sum(rate(http_requests_total{namespace="{namespace}", app="{app_label}", status=~"5.."}[5m])) / sum(rate(http_requests_total{namespace="{namespace}", app="{app_label}"}[5m])))`
- `OBS_QUERY_METRICS_RESTART_COUNT`
  - default:
    - `sum(increase(kube_pod_container_status_restarts_total{namespace="{namespace}", pod=~"{pod_pattern}.*"}[{selected_range}]))`

## Health timeline templates

- `OBS_QUERY_TIMELINE_AVAILABILITY`
  - default:
    - `avg_over_time(up{namespace="{namespace}", app="{app_label}"}[5m])`
- `OBS_QUERY_TIMELINE_ERROR_RATE`
  - default:
    - `100 * (sum(rate(http_requests_total{namespace="{namespace}", app="{app_label}", status=~"5.."}[5m])) / sum(rate(http_requests_total{namespace="{namespace}", app="{app_label}"}[5m])))`
- `OBS_QUERY_TIMELINE_READINESS`
  - default:
    - `avg_over_time(kube_deployment_status_replicas_available{namespace="{namespace}", deployment="{deployment_name}"}[5m]) / clamp_min(avg_over_time(kube_deployment_spec_replicas{namespace="{namespace}", deployment="{deployment_name}"}[5m]), 1)`

## Template variables

- `{namespace}`: Kubernetes namespace scope.
- `{app_label}`: service/app selector.
- `{selected_range}`: selected API range token (`1h`, `24h`, `7d`).
- `{pod_pattern}`: escaped pod selector prefix derived from service id.
- `{deployment_name}`: deployment selector derived from service id.

Missing variables fail request rendering with stable HTTP 422-style validation behavior.
