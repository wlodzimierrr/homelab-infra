# 0003: Secrets management strategy (SOPS + age)

- Status: Accepted
- Date: 2026-02-27
- Owners: @wlodzimierrr

## Context

The platform needs a secure, GitOps-compatible approach for managing Kubernetes secrets across environments.

The current repository uses encrypted Ansible Vault data for automation variables, but workload-level Kubernetes secret management needs a single GitOps standard.

Roadmap work calls out a decision between SOPS and Sealed Secrets before broader workload rollout.

## Decision

Use SOPS with age keys as the default secrets-as-code approach.

Implementation baseline:

1. Secrets committed to Git must be SOPS-encrypted (`*.enc.yaml`) and include a `sops` metadata block.
2. Plain Kubernetes `Secret` manifests are disallowed in `workloads/`.
3. Runtime secret creation/rotation is performed by decrypting from Git at apply time, with runbook-driven validation.

## Alternatives considered

1. SOPS (selected): file-level encryption in Git, flexible tooling, no hard dependency on a single in-cluster controller.
2. Sealed Secrets: strong Kubernetes-native workflow, but ties encryption to cluster sealing keys and adds controller dependency.
3. External secret manager first (Vault/ESO): strongest dynamic model but higher setup and operational burden for current homelab phase.

## Consequences

1. Positive: encrypted-at-rest secret workflow is now standardized across workloads.
2. Positive: Git history remains auditable while preventing plaintext credential exposure.
3. Tradeoff: operators need SOPS/age tooling and key management discipline.
4. Follow-up: complete T3.3.1 validation by proving Argo CD applies manifests that consume decrypted secret values without manual post-fixes.
