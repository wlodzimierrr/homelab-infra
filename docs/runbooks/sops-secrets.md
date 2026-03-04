# SOPS Secrets Runbook

This runbook implements T3.3.1 using SOPS + age.

## 1. Install tooling

Install `sops` and `age` on your operator workstation.

## 2. Create an age keypair

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
grep '^# public key:' ~/.config/sops/age/keys.txt
```

Save the printed public key. It is used in `.sops.yaml`.

## 3. Configure repository creation rules

Create/update `workloads/.sops.yaml`:

```yaml
creation_rules:
  - path_regex: apps/.*/envs/.*/.*secret.*\.enc\.ya?ml$
    encrypted_regex: "^(data|stringData)$"
    age:
      - age1REPLACE_WITH_YOUR_PUBLIC_KEY
```

## 4. Encrypt secret manifests

Example for API Postgres secret:

```bash
cd workloads
./scripts/bootstrap-sops-postgres-secret.sh dev
./scripts/bootstrap-sops-postgres-secret.sh prod
```

Repeat per environment and for other app secrets.

## 5. Wire encrypted secrets into overlays

Only do this after Argo CD decryption integration is configured.

Current homelab state:

1. Argo CD is not configured with SOPS decryption plugin yet.
2. Do **not** include `postgres-secret.enc.yaml` in overlay `resources`, or pods will receive literal `ENC[...]` values.
3. Keep encrypted files in Git for source-of-truth and decrypt/apply at runtime from operator workflow.

## 6. Validate guardrails

```bash
cd workloads
./scripts/check-secrets-guardrails.sh
```

This fails if plaintext `kind: Secret` manifests are committed without a `sops:` block.

## 7. Validate decryption and apply

Local check:

```bash
sops --decrypt workloads/apps/homelab-api/envs/dev/postgres-secret.enc.yaml >/dev/null
```

Cluster check (using local decryption in operator or pipeline step):

```bash
sops --decrypt workloads/apps/homelab-api/envs/dev/postgres-secret.enc.yaml | kubectl apply --dry-run=server -f -
sops --decrypt workloads/apps/homelab-api/envs/dev/postgres-secret.enc.yaml | kubectl apply -f -
```

## 8. Acceptance evidence for T3.3.1

Capture and store:

1. `git grep -n "^kind: Secret"` output showing only SOPS-managed files.
2. Guardrail script pass output.
3. `kubectl` apply/sync output confirming workloads start with decrypted secret values.
