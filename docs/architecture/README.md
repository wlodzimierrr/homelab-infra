# Architecture Diagrams

This section keeps lightweight architecture maps in Markdown and Mermaid. The diagrams follow the repo state in this workspace: manifests, app code, CI workflows, Terraform, Ansible, and runbooks. If live cluster state differs from the repo, the docs call that out explicitly instead of guessing.

## Views

- [Platform Overview](platform-overview.md)
- [Deployment Flow](deployment-flow.md)
- [Service / API Map](service-api-map.md)

## Related ADRs

- [ADR 0001: Kubernetes distro and cluster topology (K3s HA)](../adr/0001-k3s-ha-cluster-topology.md)
- [ADR 0002: GitOps controller and structure (Argo CD)](../adr/0002-argocd-gitops-model.md)
- [ADR 0004: Manifest layering strategy (Kustomize overlays over Helm values)](../adr/0004-manifest-layering-kustomize-over-helm-values.md)
- [ADR 0005: Day-0 vs Day-2 ownership model](../adr/0005-day-0-vs-day-2-ownership-model.md)
- [ADR 0006: OAuth provider vs Cloudflare Zero Trust](../adr/0006-oauth-vs-cloudflare-zero-trust.md)
- [ADR 0007: Portal-to-Git workflow model](../adr/0007-portal-to-git-workflow-model.md)

## Update Checklist

- Day-0 edge, network, or ingress changes: `terraform/gateway/`, `ansible/`, [`../day-0-rebuild-runbook.md`](../day-0-rebuild-runbook.md)
- Argo CD project or app topology changes: `workloads/bootstrap/`, `workloads/environments/`
- Workload namespace, service, or auth changes: `workloads/apps/homelab-api/`, `workloads/apps/homelab-web/`
- Monitoring and logging changes: `workloads/environments/dev/workloads/`, `workloads/platform/monitoring/`
- API integration changes: `apps/portal/backend/app/`
- Frontend routing or API proxy changes: `apps/portal/frontend/`
- Delivery pipeline changes: `apps/portal/.github/workflows/`

## Notes

- Argo CD `repoURL` fields and the portal promotion workflows currently reference `https://github.com/wlodzimierrr/homelab-workloads.git`.
- This workspace also contains a `workloads/` tree with the same manifest layout, so keep both the diagrams and the manifest source of truth aligned with the repo URL that Argo CD actually watches.
