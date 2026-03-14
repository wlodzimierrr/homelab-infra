# 0007: Portal-to-Git workflow model

- Status: Accepted
- Date: 2026-03-11
- Owners: @wlodzimierrr

## Context

The portal needs a standard way to propose Day-2 changes for deployments, promotions, and configuration without bypassing GitOps.

Existing platform decisions already constrain the solution:

- ADR 0002 makes Argo CD the reconciler and Git the desired-state source.
- ADR 0005 splits Day-0 ownership (Terraform/Ansible) from Day-2 ownership (Argo CD).
- The current image promotion workflows already open pull requests against `wlodzimierrr/homelab-workloads` instead of applying cluster changes directly.

The new portal workflows must preserve those boundaries while giving a solo operator:

- auditable change history
- predictable rollback and review points
- minimal operational overhead
- least-privilege credentials

The decision under consideration is how the portal backend creates Git changes:

- Option A: use the GitHub API directly to create branches, commits, and pull requests
- Option B: write requests to a dedicated branch and rely on a separate bot to open pull requests
- Option C: let the portal push direct commits to feature branches through generic Git operations

## Decision

Choose Option A: the portal backend will use the GitHub API directly to create managed branches, write constrained file changes, and open pull requests in the GitOps repository.

Decision details:

- All portal-initiated Day-2 changes must result in a pull request against the GitOps repo watched by Argo CD.
- The portal must not fall back to direct cluster mutation, Argo CD API writes, or `kubectl`-style deployment logic.
- The backend implementation should expose a provider-shaped interface (`GitService`) but ship with a GitHub-backed implementation first, because GitHub is the active source-control provider for both the portal and workloads repos.
- Pull requests should carry structured metadata for actor, action, environment, service, and request identifier so deployment records and audit trails can correlate one portal request to one GitOps PR.

## Implementation plan

Authentication and token scope:

- Start with a fine-grained GitHub token scoped to the single workloads repository.
- Grant only the minimum permissions required for this model:
  - `Contents`: read/write
  - `Pull requests`: read/write
  - `Metadata`: read-only
- Do not grant org admin, user profile, Actions admin, or broad multi-repo scopes.
- Store the token as `GIT_GITHUB_TOKEN` in the `homelab-api` runtime secret, encrypted at rest with SOPS.
- Keep repo owner, repo name, and default base branch configurable so the implementation is not hard-coded to one environment.

Safety boundaries:

- Restrict writes to an allow-listed set of paths per action type (deploy, promote, rollback, config change, service registration).
- Create portal-managed branches from the default branch using a deterministic naming scheme such as `portal/<action>/<service>/<request-id>`.
- Use request keys or idempotency keys so a retried portal action reuses or updates the same logical PR instead of spamming duplicates.
- Require dry-run support that renders the proposed file mutations and PR payload without calling GitHub, so tests and outage handling use the same rendering path.
- Record the resulting branch name, commit SHA, PR number, and PR URL in deployment/config audit records.

Operational behavior:

- Open pull requests immediately after the commit is created so GitHub becomes the review and approval surface.
- Keep merge policy outside the portal service; branch protection, required reviewers, and merge rules remain enforced by the GitHub repository.
- Treat GitHub as the provider-specific transport only. The portal owns request rendering and validation; GitHub owns branch, commit, and PR objects.

Fallback for GitHub outage or rate limits:

- On `403` secondary rate limits, `429`, or transient `5xx` responses, retry with bounded exponential backoff.
- If GitHub remains unavailable, persist the rendered request payload and mark the portal action as blocked by external dependency rather than mutating the cluster through another path.
- Provide operator-visible failure details plus the exact branch name, target files, commit message, and rendered diff payload needed to replay the request manually once GitHub recovers.
- Manual fallback is "create the same GitOps PR later", not "deploy around GitOps". Argo CD remains the only reconciler.

## Alternatives considered

- Option B: requests branch plus PR bot
  - Pros: can isolate GitHub write credentials into a dedicated automation component and makes future provider swapping somewhat cleaner.
  - Cons: adds a second service or workflow to maintain, introduces queue latency, splits audit context across two systems, and creates a new failure mode where the portal succeeds but the bot does not.
  - Rejected because the homelab is single-operator and the extra bot layer adds operational burden without enough payoff.
- Option C: direct Git commits to feature branches
  - Pros: simplest generic Git abstraction and potentially easier to adapt to non-GitHub providers.
  - Cons: weaker repository-level audit semantics, broader credential patterns (PAT or SSH for raw Git push), more branch-cleanup burden, and less control over constrained PR metadata.
  - Rejected because it optimizes for provider neutrality over the current need for safe, reviewable GitHub-native automation.

## Consequences

- Positive: aligns new portal workflows with the GitHub PR model already used by the existing delivery pipelines.
- Positive: every requested change has a first-class GitHub audit object that can be reviewed, linked, approved, closed, or reverted.
- Positive: keeps Argo CD as the sole deployment reconciler and avoids direct cluster-write drift.
- Tradeoff: the first implementation is GitHub-specific and depends on GitHub API availability.
- Tradeoff: a fine-grained token still represents a write credential and must be rotated, monitored, and path-constrained carefully.
- Follow-up: implement the backend Git service, add dry-run and retry behavior, document token rotation, and add an operator runbook for replaying blocked requests after GitHub outages.
