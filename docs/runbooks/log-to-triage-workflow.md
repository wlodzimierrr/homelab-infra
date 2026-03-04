# Log-to-Triage Workflow Runbook

This runbook covers T4.2.2 and defines a repeatable workflow to correlate logs with deployments and incidents.

## 1. Scope

Use this workflow when an alert fires, user-facing behavior regresses, or deployment health degrades and logs are needed to identify root cause quickly.

Data sources:

1. Loki logs (`namespace`, `app`, `container`, `pod` labels)
2. Argo CD application sync history
3. Kubernetes rollout/events/pod status

## 2. Standard triage workflow

1. Capture incident window:
   - Record `INCIDENT_START_UTC` and current UTC time.
2. Check active deploy/sync context:
   - `argocd app get homelab-api-dev --grpc-web`
   - `argocd app get homelab-web-dev --grpc-web`
3. Check workload rollout/restart signals:
   - `kubectl -n homelab-api get deploy homelab-api -o wide`
   - `kubectl -n homelab-web get deploy homelab-web -o wide`
   - `kubectl -n homelab-api get pods -o wide`
   - `kubectl -n homelab-web get pods -o wide`
4. Query logs for the same time window:
   - In Grafana Explore (Loki), filter by namespace/app/container.
   - Or use Loki API query_range.
5. Correlate:
   - Match first error timestamps against rollout starts, pod restarts, and Argo sync timestamps.
6. Classify incident:
   - Deployment-induced, configuration-induced, dependency failure, or platform-level ingestion issue.
7. Record outcome:
   - Root cause, fix, and time-to-diagnose in incident notes.

## 3. Query templates

Set a 30-minute investigation window:

```bash
START_NS=$(date -u -d '30 minutes ago' +%s%N)
END_NS=$(date -u +%s%N)
```

API logs:

```bash
curl -G -s 'http://127.0.0.1:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={namespace="homelab-api",app="homelab-api"}' \
  --data-urlencode "start=$START_NS" \
  --data-urlencode "end=$END_NS" \
  --data-urlencode 'limit=200' | jq
```

Web and oauth2-proxy logs:

```bash
curl -G -s 'http://127.0.0.1:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={namespace="homelab-web",app=~"homelab-web|oauth2-proxy"}' \
  --data-urlencode "start=$START_NS" \
  --data-urlencode "end=$END_NS" \
  --data-urlencode 'limit=200' | jq
```

Cross-workload error scan:

```bash
curl -G -s 'http://127.0.0.1:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={namespace=~"homelab-api|homelab-web"} |= "error"' \
  --data-urlencode "start=$START_NS" \
  --data-urlencode "end=$END_NS" \
  --data-urlencode 'limit=200' | jq
```

## 4. Common failure scenarios

### Scenario A: Promtail cannot push to Loki after rollout

Symptoms:

1. Promtail logs show `connect: connection refused` to Loki.
2. Loki label/query endpoints return empty or incomplete data.

Correlation steps:

1. `kubectl -n monitoring rollout status statefulset/loki --timeout=5m`
2. `kubectl -n monitoring rollout status daemonset/promtail --timeout=5m`
3. Check when Loki became Ready vs first Promtail push errors.

Likely cause:

1. Expected startup race while Loki is still initializing.

Decision:

1. If errors stop after Loki Ready and recent queries return data, close as transient.
2. If errors persist after Ready, investigate service endpoints/networking.

### Scenario B: Service unhealthy after deployment, logs missing or sparse

Symptoms:

1. Argo app shows recent sync or rollout.
2. Service endpoint fails, but expected logs are not visible in Loki query.

Correlation steps:

1. Verify recent window and label filters:
   - `namespace`, `app`, `container`.
2. Check pod/container names after rollout (labels may differ).
3. Force fresh activity:
   - `kubectl -n homelab-api rollout restart deploy/homelab-api`
   - `kubectl -n homelab-web rollout restart deploy/homelab-web`
4. Re-query immediately with updated time window.

Likely cause:

1. Query targeted stale labels/time range, not current pods/containers.

Decision:

1. Update query filters and incident template to include `app` plus `container`.

### Scenario C: Authentication or dependency failures spike

Symptoms:

1. API or DB logs show repeated auth failures.
2. User flows fail intermittently or consistently.

Correlation steps:

1. Query API + Postgres logs around first failure timestamp.
2. Compare with latest secret rotation, deployment revision, and pod restart times.
3. Validate live secret references and mounted env sources in deployment specs.

Likely cause:

1. Credential mismatch (app secret value differs from backend dependency).

Decision:

1. Rotate/align credentials and restart affected workloads.
2. Confirm error volume drops in subsequent 10-15 minutes.

## 5. Time-to-diagnose baseline (measured once)

Baseline drill:

1. Date: `2026-03-04`
2. Scenario: `Scenario A` (Promtail push failures + early Loki startup)
3. Start signal: first Promtail `connect: connection refused` warning at `2026-03-04T21:52:36Z`
4. Diagnosis complete: ingestion confirmed via Loki label/value queries and workload log streams
5. Time-to-diagnose: `14 minutes`

Baseline target for future drills: keep median triage time under `15 minutes` for known patterns.
