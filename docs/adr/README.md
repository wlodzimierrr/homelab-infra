# Architecture Decision Records

This folder stores Architecture Decision Records (ADRs) for the homelab.

## How to add an ADR

1. Copy [`_template.md`](_template.md).
2. Name the file `NNNN-short-title.md` (example: `0004-example.md`).
3. Fill in status, context, decision, alternatives, and consequences.
4. Add the ADR to the index below.

## Index

- [0001: Kubernetes distro and cluster topology (K3s HA)](0001-k3s-ha-cluster-topology.md)
- [0002: GitOps controller and structure (Argo CD)](0002-argocd-gitops-model.md)
- [0003: Secrets management strategy (SOPS + age)](0003-secrets-management-strategy-placeholder.md)
- [0004: Manifest layering strategy (Kustomize overlays over Helm values)](0004-manifest-layering-kustomize-over-helm-values.md)
- [0005: Day-0 vs Day-2 Ownership Model](0005-day-0-vs-day-2-ownership-model.md)
- [0006: OAuth provider vs Cloudflare Zero Trust](0006-oauth-vs-cloudflare-zero-trust.md)
