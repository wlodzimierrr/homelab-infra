# 0005: Day-0 vs Day-2 Ownership Model

- Status: Accepted
- Date: 2026-02-28
- Owners: @wlodzimierrr

## Context

The homelab platform uses multiple infrastructure provisioning tools:

- Terraform (gateway infrastructure)
- Ansible (cluster bootstrap and host configuration)
- Argo CD (GitOps-based application reconciliation)

As the platform grows (observability stack, security tooling, registry integration, internal workloads), it becomes necessary to clearly define:

- Which system owns initial provisioning
- Which system owns steady-state reconciliation
- Which repository is the source of truth for each layer (including `workloads` for Day-2 app and platform manifests)

Without explicit boundaries, there is risk of:

- Configuration drift
- Duplicate deployment logic
- Confusion between bootstrap and runtime ownership
- Inconsistent lifecycle management

## Decision

Define a strict ownership boundary between Day-0 and Day-2.

### Day-0 (Provisioning Layer)

Owned by:

- Terraform
- Ansible

Responsibilities:

- External infrastructure provisioning (gateway, networking)
- Node configuration and OS-level setup
- Kubernetes cluster bootstrap (k3s HA)
- Installation of ingress controller
- Installation of cert-manager
- Installation of Argo CD
- Application of initial Argo CD root Application

Characteristics:

- Executed manually or via CI
- Idempotent but not continuously reconciling
- Required to bring cluster to operational baseline
- Safe to re-run for disaster recovery

### Day-2 (Reconciliation Layer)

Owned by:

- Argo CD

Responsibilities:

- Deployment of platform add-ons (observability, logging, image automation, etc.)
- Deployment of workloads
- Namespace management (post-bootstrap)
- Continuous drift correction
- Declarative environment promotion

Characteristics:

- Continuous reconciliation
- Git as single source of truth
- No manual `kubectl apply` in steady-state
- Environment-specific path-based applications

### Architectural Boundary Rule

If a component must be:

- Continuously reconciled
- Upgradable via Git commit
- Observable via Argo CD sync status
- Managed per-environment

It belongs in Day-2 and must be deployed via Argo CD.

If a component must exist before Argo CD can function, or configures host-level infrastructure:

It belongs in Day-0 and is provisioned via Ansible or Terraform.

## Examples

Day-0 components:

- Hetzner gateway infrastructure
- WireGuard setup
- k3s HA cluster
- MetalLB installation
- Traefik installation
- cert-manager bootstrap
- Argo CD installation
- Argo CD root Application bootstrap

Day-2 components:

- Prometheus + Grafana
- Loki
- Argo Image Updater
- Internal workloads (homelab-api, homelab-web)
- Environment-specific applications
- Future self-service provisioning controllers

## Alternatives considered

- Use Ansible for everything:
  - Pros: tooling consistency; single execution model.
  - Cons: no continuous reconciliation; no drift detection; harder upgrades and rollbacks; duplicates GitOps control plane.
  - Rejected due to violation of GitOps principles.
- Use Argo CD for everything (including bootstrap):
  - Pros: single control plane; pure GitOps.
  - Cons: chicken-and-egg problem; complex initial bootstrapping; harder disaster recovery without external provisioning layer.
  - Rejected due to operational risk and reduced bootstrap reliability.

## Consequences

- Positive: clear separation of concerns.
- Positive: reduced tool overlap.
- Positive: cleaner disaster recovery story.
- Positive: predictable lifecycle management.
- Positive: production-aligned architecture model.
- Tradeoff: requires discipline to avoid deploying Day-2 components via Ansible.
- Tradeoff: two toolchains must be maintained.
- Follow-up: audit existing platform components against the Day-0/Day-2 boundary and move outliers to the correct layer.
