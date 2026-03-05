# Service Registry Diagnostics Endpoint

Use this endpoint to inspect registry freshness and release-join mismatch health.

## Endpoint

```http
GET /service-registry/diagnostics?env=dev
```

## Response fields

- `freshness.rowCount`
- `freshness.lastSyncedAt`
- `freshness.staleAfterMinutes`
- `freshness.isEmpty`
- `freshness.isStale`
- `freshness.state` (`fresh` | `stale` | `empty`)
- `joinMismatch.ciUnmatchedCount`
- `joinMismatch.argoUnmatchedCount`
- `joinMismatch.ciUnmatchedKeys` (`serviceId|serviceName|env`)
- `joinMismatch.argoUnmatchedKeys` (`serviceId|serviceName|env`)

## Smoke check examples

```bash
curl -sS -H 'Authorization: Bearer dev-static-token' \
  'http://api.dev.homelab.local/service-registry/diagnostics?env=dev' | jq
```

Alert conditions to watch:

- `freshness.state == "stale"`: sync likely not running or blocked
- `freshness.state == "empty"` with expected services: registry bootstrap/sync issue
- `joinMismatch.*Count > 0`: upstream release metadata keys not aligning with registry projection

