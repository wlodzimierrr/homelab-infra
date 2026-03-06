# OIDC/SSO Setup Runbook

This runbook configures centralized SSO for both Argo CD and Portal ingress.

## 1. Prepare GitHub OAuth apps

Create two GitHub OAuth apps:

1. Argo CD app callback: `https://argocd.wlodzimierrr.co.uk/api/dex/callback`
2. Portal app callback: `http://portal.dev.homelab.local/oauth2/callback`

## 2. Configure Argo CD Dex (GitHub) and RBAC

Create/update Argo CD secret out-of-band (do not commit credentials):

```bash
kubectl -n argocd create secret generic argocd-secret \
  --from-literal=dex.github.clientID='<ARGOCD_GH_CLIENT_ID>' \
  --from-literal=dex.github.clientSecret='<ARGOCD_GH_CLIENT_SECRET>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Apply:

```bash
kubectl apply -f workloads/bootstrap/argocd-oidc.yaml
kubectl -n argocd rollout restart deployment argocd-server
```

This manifest includes:

1. Dex GitHub connector settings in `argocd-cm`
2. Group-based role mappings in `argocd-rbac-cm`
3. `admin.enabled: "false"` to keep local admin disabled

## 3. Configure portal oauth2-proxy forward-auth (dev)

Create/update oauth2-proxy secret out-of-band (do not commit credentials):

```bash
kubectl -n homelab-web create secret generic oauth2-proxy-secret \
  --from-literal=OAUTH2_PROXY_CLIENT_ID='<PORTAL_GH_CLIENT_ID>' \
  --from-literal=OAUTH2_PROXY_CLIENT_SECRET='<PORTAL_GH_CLIENT_SECRET>' \
  --from-literal=OAUTH2_PROXY_COOKIE_SECRET="$(openssl rand -base64 32)" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Then apply the dev overlay:

```bash
kubectl apply -k workloads/apps/homelab-web/envs/dev
```

The dev overlay adds:

1. `oauth2-proxy` Deployment/Service/Secret
2. Traefik `oauth2-errors` + `oauth2-forward-auth` middleware
3. Ingress patch for `/oauth2/*` and forward-auth on both UI and `/api/*`
4. NetworkPolicies for oauth2-proxy ingress and egress

## 4. Map claims to app roles

Portal backend accepts oauth2-proxy identity headers:

1. `X-Auth-Request-User`
2. `X-Auth-Request-Groups`

Admin actions (`POST /service-registry/sync`) allow:

1. Any user in `PORTAL_ADMIN_USERS` (default `admin`)
2. Any group in `PORTAL_ADMIN_GROUPS` (default `team-admins`)

Set these env vars in API deployment per environment as needed.

## 5. Validation checks

Argo CD claim mapping:

```bash
argocd login argocd.wlodzimierrr.co.uk --sso
argocd account get-user-info
```

Confirm `groups` contains the expected team and that resulting role matches `argocd-rbac-cm`.

Portal auth and roles:

```bash
curl -i http://portal.dev.homelab.local/
curl -i -X POST 'http://portal.dev.homelab.local/api/service-registry/sync?source=gitops_apps&env=dev' \
  -H 'X-Auth-Request-User: alice' \
  -H 'X-Auth-Request-Groups: team-readers'
```

Expected:

1. Unauthenticated browser request redirects to oauth2-proxy login flow.
2. Non-admin group receives `403` on catalog sync.
3. Admin group receives `200` on catalog sync.

## 6. Break-glass

Use the procedure in [sso-break-glass.md](sso-break-glass.md).
