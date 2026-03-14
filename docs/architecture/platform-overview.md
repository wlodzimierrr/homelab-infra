# Platform Overview

This view summarizes the repo-defined runtime platform: the day-0 gateway and cluster edge, Argo CD GitOps wiring, workload namespaces, and the main portal and observability components.

```mermaid
flowchart TB
  subgraph Edge["Public edge and external systems"]
    Internet["Internet / remote operator"]
    Gateway["Hetzner gateway VM\nTerraform-managed"]
    GitHub["GitHub\nrepo hosting + OAuth identity"]
    GitOpsRepo["GitOps repo URL\nwlodzimierrr/homelab-workloads"]
  end

  subgraph Network["Private network path"]
    DNAT["Public 80/443 DNAT"]
    WG["WireGuard route\n10.100.0.0/24 overlay"]
  end

  subgraph Cluster["Home LAN k3s HA cluster"]
    MetalLB["MetalLB\nVIP pool 192.168.50.240-250"]
    Traefik["Traefik ingress\nkube-system"]
    CertMgr["cert-manager"]

    subgraph ArgoNS["argocd namespace"]
      Argo["Argo CD\nroot + child apps\nargocd.wlodzimierrr.co.uk"]
      Dex["Dex GitHub connector"]
    end

    subgraph PlatformNS["platform namespace"]
      PlatformStub["Namespace scaffold\nminimal repo-managed resources today"]
    end

    subgraph WebNS["homelab-web namespace"]
      Web["homelab-web\nReact SPA served by Nginx\nportal.dev.homelab.local\nportal.homelab.local"]
      OAuth["oauth2-proxy\n(dev overlay only)"]
    end

    subgraph ApiNS["homelab-api namespace"]
      API["homelab-api\nFastAPI\napi.dev.homelab.local\napi.homelab.local"]
      PG["homelab-api-postgres\nStatefulSet + PVC"]
      Migrate["homelab-api-migrate\nArgo sync-hook Job"]
      Sync["homelab-api-catalog-sync\nCronJob"]
    end

    subgraph MonNS["monitoring namespace"]
      Prom["kube-prometheus-stack\nPrometheus"]
      Alert["Alertmanager\nrepo manifest state"]
      Grafana["Grafana\nmonitoring.homelab.local"]
      Loki["Loki"]
      Promtail["Promtail"]
    end
  end

  Internet --> Gateway --> DNAT --> WG --> MetalLB --> Traefik
  GitHub --> Dex
  GitHub --> OAuth
  GitOpsRepo --> Argo

  CertMgr --> Argo
  Traefik --> Argo
  Traefik -. ForwardAuth .-> OAuth
  Traefik --> Web
  Traefik --> API
  Traefik --> Grafana

  Argo --> PlatformStub
  Argo --> Web
  Argo --> API
  Argo --> Prom
  Argo --> Loki
  Argo --> Promtail
  Argo --> Grafana

  API --> PG
  Migrate --> PG
  Sync --> GitOpsRepo
  Prom --> Grafana
  Loki --> Grafana
  Promtail --> Loki
```

## What It Shows

The repo currently models a split between day-0 infrastructure and day-2 GitOps workloads. Terraform and Ansible establish the Hetzner gateway, WireGuard path, k3s HA cluster, MetalLB, Traefik, cert-manager, and Argo CD. Argo CD then reconciles the workload namespaces for `homelab-web`, `homelab-api`, and `monitoring`.

The `platform` namespace appears to be mostly a scaffold in this repo today, rather than a large bundle of shared services. That is inferred from `workloads/platform/base/kustomization.yaml` and the current platform app contents.

Alertmanager is shown because the current dev `kube-prometheus-stack` values in `workloads/environments/dev/workloads/monitoring-app.yaml` set `alertmanager.enabled: true`. Some runbooks and issue notes still describe the older disabled state, so treat this as "repo manifest state; cluster validation may still be pending".

## Trust Boundaries

- The public edge ends at the Hetzner gateway. Public `80/443` traffic is forwarded across a WireGuard path into the home LAN before it reaches Traefik.
- GitHub is outside the cluster trust boundary for both repo hosting and OAuth identity. Argo CD GitOps access and the dev portal OAuth flow both depend on it.
- Namespace boundaries matter inside the cluster. `homelab-web` and `homelab-api` both use default-deny NetworkPolicies with explicit allow lists, while monitoring runs in its own namespace and is queried in-cluster by the backend.

## Update It When

- Gateway, WireGuard, MetalLB, or Traefik routing changes in `terraform/gateway/`, `ansible/inventory/`, or [`../day-0-rebuild-runbook.md`](../day-0-rebuild-runbook.md)
- Argo CD bootstrap or project structure changes in `workloads/bootstrap/` or `workloads/environments/`
- Namespace contents change under `workloads/apps/homelab-api/`, `workloads/apps/homelab-web/`, or `workloads/environments/dev/workloads/`
- Auth decisions move between basic auth, oauth2-proxy, Dex, or another provider; see [ADR 0006](../adr/0006-oauth-vs-cloudflare-zero-trust.md)
