# SSO Break-Glass Runbook

Use this only during identity provider or oauth2-proxy outages.

## Preconditions

1. Incident is declared (auth outage blocks operator access).
2. Change is tracked in ticket/incident notes.

## Path A: Emergency VPN access (preferred)

1. Connect through WireGuard.
2. Access Argo CD or services over internal network paths.
3. Perform required recovery actions.

## Path B: Temporary ingress bypass for portal

Temporarily remove oauth2 middlewares from dev ingress:

```bash
kubectl -n homelab-web annotate ingress homelab-web \
  traefik.ingress.kubernetes.io/router.middlewares-
kubectl -n homelab-web annotate ingress homelab-web-api \
  traefik.ingress.kubernetes.io/router.middlewares-
```

Restore SSO middlewares after recovery:

```bash
kubectl -n homelab-web annotate ingress homelab-web \
  traefik.ingress.kubernetes.io/router.middlewares=homelab-web-oauth2-errors@kubernetescrd,homelab-web-oauth2-forward-auth@kubernetescrd \
  --overwrite
kubectl -n homelab-web annotate ingress homelab-web-api \
  traefik.ingress.kubernetes.io/router.middlewares=homelab-web-oauth2-errors@kubernetescrd,homelab-web-oauth2-forward-auth@kubernetescrd \
  --overwrite
```

## Path C: Argo CD local admin recovery (time-boxed)

1. Re-enable local admin only for recovery window.
2. Rotate credentials immediately after use.
3. Re-disable local admin and confirm SSO login works.

## Post-recovery validation

1. SSO login works for Argo CD and portal.
2. Break-glass changes are reverted.
3. GitOps state matches cluster (commit/revert any emergency manifest change).
