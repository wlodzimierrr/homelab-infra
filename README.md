# Homelab Platform

A **production-grade Infrastructure-as-Code platform** for a personal Kubernetes homelab, demonstrating enterprise cloud-native architecture patterns: GitOps with Argo CD, declarative infrastructure management, multi-layered secrets handling, and comprehensive observability.

## Project Overview

This is a **complete DevOps platform** combining:
- **Day-0 Infrastructure**: Hetzner cloud gateway, WireGuard VPN, Terraform IaC
- **Day-1 Cluster Setup**: K3s HA control plane, MetalLB, cert-manager, Traefik ingress with automated TLS
- **Day-2 Operations**: GitOps automation via Argo CD, workload deployment, secrets management via SOPS/age
- **Applications**: Portal (React SPA + FastAPI backend), observability stack (Prometheus, Grafana, Loki)
- **Platform Engineering**: Service catalog, RBAC, API contracts, deployment hooks, audit trails

**Target Audience**: Solo infrastructure operator with CI/CD automation, auditable change tracking, and least-privilege security.

---

## Related Repositories

| Repository | Purpose | Status |
|---|---|---|
| **[wlodzimierrr/homelab](https://github.com/wlodzimierrr/homelab)** | Day-0 IaC, Ansible roles, portal app source | This repo |
| **[wlodzimierrr/homelab-workloads](https://github.com/wlodzimierrr/homelab-workloads)** | GitOps manifests, Kustomize overlays, deployments | Watched by Argo CD |
| **[wlodzimierrr/homelab-web](https://github.com/wlodzimierrr/homelab-web)** | React SPA frontend (portal UI) | Deployed to cluster |
| **[wlodzimierrr/homelab-api](https://github.com/wlodzimierrr/homelab-api)** | FastAPI backend (service catalog, scaffold, deployments) | Deployed to cluster |

---

## Project Structure

```
homelab/                                # Day-0 IaC + portal monorepo
├── terraform/                          # Hetzner gateway, networking, firewall
│   └── gateway/                        # Terraform modules
├── ansible/                            # K3s cluster bootstrap, system config
│   ├── roles/                          # k3s_ha, metallb, cert_manager, argocd, traefik, wireguard
│   ├── playbooks/                      # Orchestration (k3s_ha.yml, argocd.yml, etc.)
│   └── inventory/                      # Hosts, group vars, host vars
├── apps/                               # Portal monorepo
│   └── portal/
│       ├── backend/                    # FastAPI (Python 3.10+)
│       │   ├── app/api/                # Route handlers, schemas, dependencies
│       │   ├── app/services/           # Domain logic (ScaffoldService, DeployService, etc.)
│       │   └── tests/                  # Pytest suite
│       └── frontend/                   # React SPA
├── workloads/                          # GitOps manifests (template, also in homelab-workloads)
│   ├── bootstrap/                      # Argo CD root app + child app definitions
│   ├── environments/                   # dev/ + prod/ overlay paths
│   ├── apps/                           # Service deployments (homelab-api, homelab-web)
│   ├── platform/                       # Shared: monitoring, ingress, operators
│   └── services.yaml                   # Service catalog
├── docs/                               # Architecture, runbooks, contracts
│   ├── architecture/                   # Platform overview, deployment flow, API map
│   ├── adr/                            # Architecture Decision Records
│   ├── contracts/                      # API contracts, service identity, observability
│   ├── runbooks/                       # Operational guides
│   └── monitoring/                     # Prometheus queries, alert dashboards
└── README.md                           # This file
```

---

## Key Architecture Decisions

Documented via Architecture Decision Records (ADRs) in [docs/adr/](docs/adr/):

| ADR | Decision | Rationale |
|---|---|---|
| **[0001](docs/adr/0001-k3s-ha-cluster-topology.md)** | K3s HA cluster (3 control + 2 workers) | Lightweight, HA-ready, cost-efficient |
| **[0002](docs/adr/0002-argocd-gitops-model.md)** | Argo CD GitOps (watches homelab-workloads repo) | Declarative, auditable, separation of concerns |
| **[0004](docs/adr/0004-manifest-layering-kustomize-over-helm-values.md)** | Kustomize overlays (not Helm values) | Precise control, transparency, testability |
| **[0005](docs/adr/0005-day-0-vs-day-2-ownership-model.md)** | Day-0: Terraform/Ansible \| Day-2: Argo CD | Clear ownership, reduces operational complexity |
| **[0007](docs/adr/0007-portal-to-git-workflow-model.md)** | Portal → GitHub API → Pull Requests | GitOps-first, audit trail, least-privilege |

---

## Core Components

### 1. **Gateway & Networking** (Terraform + Hetzner)
- Public edge VM with port forwarding (80/443 DNAT)
- WireGuard overlay network (10.100.0.0/24)
- See: [terraform/gateway/](terraform/gateway/)

### 2. **Kubernetes Cluster** (Ansible + K3s HA)
- 3-node control plane + 2 workers
- MetalLB VIP pool (192.168.50.240–250)
- Traefik ingress + cert-manager (automated TLS via Let's Encrypt)
- See: [ansible/roles/k3s_ha/](ansible/roles/k3s_ha/)

### 3. **GitOps Platform** (Argo CD)
- Root app monitors `wlodzimierrr/homelab-workloads` (production manifests)
- Child apps per workload namespace
- KSOPS + SOPS/age for secret decryption at sync time
- See: [workloads/bootstrap/](workloads/bootstrap/)

### 4. **Portal Application** (React + FastAPI)
- **Frontend**: React SPA served by Nginx
  - Dev: `portal.dev.homelab.local`
  - Prod: `portal.homelab.local`
  - Repo: [wlodzimierrr/homelab-web](https://github.com/wlodzimierrr/homelab-web)
- **Backend**: FastAPI service catalog + deployment automation
  - Dev: `api.dev.homelab.local`
  - Prod: `api.homelab.local`
  - Repo: [wlodzimierrr/homelab-api](https://github.com/wlodzimierrr/homelab-api)
  - Postgres StatefulSet + migrations via sync hooks
  - CronJob for catalog sync from Git

### 5. **Observability Stack**
- **Prometheus**: cluster metrics + application metrics
- **Grafana**: dashboards + alerting
- **Loki**: log aggregation (promtail sidecars)
- Alert manager rules in [workloads/platform/monitoring/](workloads/platform/monitoring/)

---

## Documentation Index

| Document | Purpose |
|---|---|
| **[Architecture Overview](docs/architecture/README.md)** | Platform diagrams (Mermaid), component interactions, data flow |
| **[Platform Rebuild Runbook](docs/day-0-rebuild-runbook.md)** | Step-by-step cluster initialization from bare metal |
| **[ADR Index](docs/adr/README.md)** | All architectural decisions |
| **[Service Contracts](docs/contracts/)** | API specs, identity schemas, observability signals |
| **[Runbooks](docs/runbooks/)** | Operational procedures (add nodes, deploy workloads, troubleshoot) |
| **[Roadmap & Status](docs/homelab.md)** | Current features, TODO items, known issues |

---

## Getting Started

### Prerequisites
- Hetzner Cloud account (for gateway VM)
- Local Terraform + Ansible installed
- SSH key for cluster access
- GitHub token (fine-grained, for GitOps repo)

### Quick Start
1. **Review the architecture**: [docs/architecture/platform-overview.md](docs/architecture/platform-overview.md)
2. **Bootstrap Day-0**: [docs/day-0-rebuild-runbook.md](docs/day-0-rebuild-runbook.md)
3. **Inspect Ansible roles**: [ansible/roles/](ansible/roles/)
4. **Deploy via GitOps**: Point Argo CD to [wlodzimierrr/homelab-workloads](https://github.com/wlodzimierrr/homelab-workloads)
5. **Access the portal**: `https://portal.homelab.local` (after k3s bootstraps)

### Local Development
- Portal backend: [apps/portal/backend/README.md](apps/portal/backend/README.md)
- Portal frontend: [apps/portal/frontend/README.md](apps/portal/frontend/README.md)
- Workloads manifests: [workloads/README.md](workloads/README.md)

---

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Cloud Provider** | Hetzner Cloud | IaaS (gateway VM, snapshots, networking) |
| **IaC** | Terraform | Gateway and network provisioning |
| **Configuration Mgmt** | Ansible | Cluster bootstrap, system configuration |
| **Container Orchestration** | Kubernetes (K3s HA) | Workload orchestration |
| **Ingress** | Traefik | HTTP/HTTPS routing, load balancing |
| **Cert Management** | cert-manager | Automated TLS via Let's Encrypt |
| **Load Balancing** | MetalLB | On-prem VIP provisioning |
| **GitOps** | Argo CD | Declarative deployments, sync hooks |
| **Secret Encryption** | SOPS + age | At-rest encryption, GitOps-safe |
| **Application Framework** | FastAPI + React | Portal backend + frontend |
| **Database** | PostgreSQL | Portal state + service catalog |
| **Monitoring** | Prometheus + Grafana | Metrics collection and visualization |
| **Logging** | Loki + Promtail | Log aggregation and querying |
| **VPN** | WireGuard | Secure remote access overlay |

---

## Key Features

✅ **Declarative Infrastructure**: All resources defined as code (Terraform, Ansible, Kubernetes manifests)
✅ **GitOps Automation**: Argo CD ensures cluster state matches Git; audit trail for all changes
✅ **High Availability**: K3s HA control plane, multi-node workers, persistent storage via PVCs
✅ **Security**: SOPS/age encryption, least-privilege GitHub token, WireGuard VPN, RBAC
✅ **Observability**: Prometheus metrics, Grafana dashboards, Loki logs, alerting
✅ **Sealed Secrets**: Encrypted secrets stored in Git, decrypted at runtime via KSOPS
✅ **Multi-Environment**: dev/ + prod/ overlays in Kustomize, separate domains per environment
✅ **Service Catalog**: Portal UI for discovering, deploying, and managing services
✅ **API-First**: FastAPI backend with OpenAPI docs, service-to-service integration patterns
✅ **Deployment Automation**: Portal backend creates pull requests via GitHub API → Argo CD syncs

---

## Testing & CI/CD

- **Portal Backend Tests**: Pytest suite in `apps/portal/backend/tests/`
- **GitHub Actions Workflows**: `.github/workflows/` (image build, push to GHCR, promotion)
- **Manual Integration Tests**: Postman collection in [docs/postman/](docs/postman/)

---

## Operational Practices

- **Runbooks**: See [docs/runbooks/](docs/runbooks/) for operational procedures
- **Secrets Rotation**: SOPS/age keys managed via `workloads/.sops.yaml`
- **Backup & Restore**: Use cluster snapshots + PVC backups (manual or automated via external controller)
- **Monitoring & Alerting**: Prometheus scrape configs + Alertmanager rules in [workloads/platform/monitoring/](workloads/platform/monitoring/)

---

## Deployment Flow

```
Developer commits → GitHub
              ↓
CI/CD builds image → GHCR
              ↓
Manual promotion via portal → GitHub API
              ↓
Pull Request created (homelab-workloads repo)
              ↓
Merge (manual or auto-approved)
              ↓
Argo CD detects change → Sync
              ↓
Kustomize renders manifests
              ↓
KSOPS decrypts secrets (SOPS/age)
              ↓
Kubectl apply → Deployment
              ↓
Metrics + logs streamed to observability stack
```

---

## Contributing & Extending

1. **New Ansible role?** Create in [ansible/roles/](ansible/roles/) with standard layout
2. **New service?** Add to [workloads/services.yaml](workloads/services.yaml), create Kustomize base + overlays
3. **New portal feature?** Implement in FastAPI backend [apps/portal/backend/](apps/portal/backend/), update frontend [apps/portal/frontend/](apps/portal/frontend/)
4. **Architecture decision?** Create ADR in [docs/adr/](docs/adr/) using [template](docs/adr/_template.md)

---

## Author

**Wlodzimierz (wlodzimierrr)** — Solo infrastructure operator, platform engineer, DevOps practitioner

---

## License

[MIT](LICENSE) (or specify your license)
