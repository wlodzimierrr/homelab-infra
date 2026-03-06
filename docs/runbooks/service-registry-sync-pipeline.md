# Service Registry Sync Pipeline

This runbook validates backend service-registry sync (`T4.6.2`).

## Scope

- Reads live Kubernetes Deployments from configured namespaces.
- Optionally reads Argo CD Applications for `argoAppName` hints.
- Upserts canonical rows into `service_registry` with idempotent conflict handling.

## Trigger Sync

Admin-authenticated API trigger:

```bash
curl -sS -X POST \
  -H 'Authorization: Bearer dev-static-token' \
  http://api.dev.homelab.local/service-registry/sync | jq
```

Local script trigger:

```bash
cd apps/portal/backend
source .venv/bin/activate
python scripts/sync_service_registry.py
```

## Expected Response Fields

- `correlationId`
- `env`
- `namespaces`
- `discovered`
- `upserted`
- `inserted`
- `updated`
- `sourceFailures`
- `generatedAt`
- `durationMs`

## Idempotency Check

Run sync twice and verify second run mostly reports updates/zero net inserts:

```bash
curl -sS -X POST -H 'Authorization: Bearer dev-static-token' http://api.dev.homelab.local/service-registry/sync | jq
curl -sS -X POST -H 'Authorization: Bearer dev-static-token' http://api.dev.homelab.local/service-registry/sync | jq
```

## Services API Validation

After a successful sync, confirm the live catalog is available through the backend:

```bash
curl -sS -H 'Authorization: Bearer dev-static-token' 'http://api.dev.homelab.local/services?env=dev' | jq
curl -sS -H 'Authorization: Bearer dev-static-token' 'http://api.dev.homelab.local/services/homelab-api?env=dev' | jq
curl -sS -H 'Authorization: Bearer dev-static-token' 'http://api.dev.homelab.local/catalog/reconciliation?env=dev' | jq
curl -sS -H 'Authorization: Bearer dev-static-token' 'http://api.dev.homelab.local/service-registry/diagnostics?env=dev' | jq
```

## Source Failure Visibility

If Kubernetes/Argo source queries fail, the sync should still return a response with:

- populated `correlationId`
- non-empty `sourceFailures`
- backend logs containing `service_registry_sync_source_error`
