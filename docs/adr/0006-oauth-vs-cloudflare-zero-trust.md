---
title: "0006 - OAuth provider vs Cloudflare Zero Trust"
date: 2026-03-04
status: Accepted
author: "wlodzimierrr"
---

## Context

The platform needs a production-ready external auth model for Argo CD, the Portal frontend, and workload ingress. Two options are under consideration:

- Self-managed OAuth provider (examples: GitHub OAuth/OIDC, Keycloak, Auth0)
- Cloudflare Zero Trust (Cloudflare Access)

Decision factors: operational burden, identity source compatibility (GitHub, Google, enterprise IdP), ingress integration (Traefik/ingress controllers, Argo CD SSO), offline/local dev access, cost/privacy, and recovery/break-glass.

## Decision

We choose: Use an OAuth-based SSO (start with GitHub OAuth/OIDC for SSO) as the primary path, with Cloudflare Zero Trust documented as an acceptable alternative when external Cloudflare-managed routing is required.

Rationale summary:

- Operational burden: Using a hosted identity provider (GitHub OAuth for initial rollout) minimizes maintenance compared to self-hosting Keycloak, while giving predictable integration with CI and GitHub identities. Cloudflare Zero Trust offloads auth to Cloudflare but couples availability and routing to Cloudflare and can complicate local/dev access.
- Identity source: GitHub OAuth maps directly to the developer identity used for CI and repo permissions (single identity source for a solo operator). If enterprise IdPs are required later, Keycloak (self-hosted) or an upstream OIDC provider can be introduced and mapped to the same flows.
- Ingress integration: Traefik + oauth2-proxy (or Traefik ForwardAuth) works well for OAuth flows and is straightforward to configure. Argo CD supports OIDC/OAuth providers for SSO and can be integrated with the same provider. Cloudflare Access requires routing through Cloudflare and is simple for externally routable services but provides less direct control over identity claims inside the cluster.

## Integration plan (high level)

Goals: secure Argo CD and Portal ingress with SSO; preserve a documented and tested fallback (break-glass) method for operator access.

Phase 0 — PoC (1–2 days)
- Implement GitHub OAuth via `oauth2-proxy` behind Traefik for the Portal ingress (dev overlay). Validate login, token propagation, and that Authorization headers reach backend when needed.
- Integrate Argo CD with GitHub as an OIDC provider (or use Dex/Keycloak if claim-mapping needed). Test user/group claim mapping and role enforcement.

Phase 1 — Productionize (3–5 days)
- Harden `oauth2-proxy` deployment: HTTPS cookies, secure cookie flags, session TTL, refresh behavior, and CSRF protections.
- Create `argocd` OIDC client + map groups/claims to Argo CD roles. Add automated tests for role mapping.
- Update Portal to prefer SSO login and fall back to the existing JWT login for service-to-service auth.

Phase 2 — Options / future work
- If group/claim mapping requirements or offline SSO are needed, evaluate self-hosted Keycloak and migrate OIDC clients. Keycloak provides richer mapping and on-prem control at the cost of extra maintenance.
- If most traffic is routed via Cloudflare and a low-ops model is preferred, consider adopting Cloudflare Zero Trust for ingress-level gating and use it as the primary ingress authenticator. Keep OIDC provider for Argo CD to preserve fine-grained role mapping.

Fallback / break-glass methods

- Emergency VPN access (preferred): use existing WireGuard gateway to reach cluster API and internal services for emergency operator access. Document the exact WireGuard profile rotation and access checklist in the runbook.
- Ingress break-glass: maintain a short-lived basic-auth path (disabled by default) that can be enabled via SOPS-encrypted secret toggle and documented restoration steps. Keep the toggle reviewable in Git and gated by approval.
- Admin bypass: disable anytime admin bypass is off in regular operation; if needed for recovery, re-enable a pre-shared SOPS-encrypted admin token which is rotated immediately after use.

Acceptance criteria mapping

- Decision ADR accepted: this document (this ADR) fulfills that criterion.
- Integration plan includes fallback access: See "Fallback / break-glass methods" above.

## Alternatives considered

- Cloudflare Zero Trust: Pros — low ops, built-in Identity/Access rules, device posture checks; Cons — external dependency on Cloudflare for auth and routing, harder local/dev usage, potential cost and privacy considerations.
- Self-hosted Keycloak: Pros — full control, flexible claim/group mapping; Cons — higher operational overhead (updates, backups, availability).

## Consequences

- Short term: Use GitHub OAuth for reduced ops and fast integration with Argo CD and Portal. Expect to implement `oauth2-proxy` + Traefik ForwardAuth and OIDC configuration in Argo CD.
- Long term: Keep paths open to migrate to Cloudflare ZT or Keycloak if requirements change (enterprise IdP, device posture, or outsourcing auth).

## Next steps

1. Create PoC manifests for `oauth2-proxy` + Traefik (dev overlay).
2. Configure Argo CD OIDC client and test role mapping.  
3. Document WireGuard break-glass runbook and add SOPS-encrypted basic-auth toggle for ingress.
4. If desired, evaluate Cloudflare Zero Trust with a small workload to confirm dev/local implications.

---

References:
- T3.2.1 in the project roadmap
- Traefik + oauth2-proxy forward-auth patterns
- Argo CD OIDC documentation
