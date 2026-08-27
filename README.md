# healfmx-pr-agent-settings

Centralized PR-Agent configuration and reusable GitHub Actions workflow for
AI-powered automated Pull Request review (Claude Sonnet 5) across healfmx
repos.

## What lives here

- `.pr_agent.toml` — the single source of truth for review behavior (model,
  commands run, cost tracking). Do not fork per-repo copies.
- `.github/workflows/pr-review-reusable.yml` — the reusable workflow every
  consuming repo calls. Handles AWS OIDC role assumption, Secrets Manager
  lookup, and the PR-Agent action invocation.

## One-time setup (AWS + GitHub, per account — not per repo)

This account is an individual GitHub account, not an Organization, so there
is no org-wide secrets store. Each consuming repo needs its own two repo
secrets pointing at the *same* shared AWS resources below.

1. **AWS Secrets Manager** — create one secret holding the Anthropic API key:
   ```
   Name:  healfmx/pr-review/anthropic-api-key
   Value: {"ANTHROPIC_API_KEY": "sk-ant-..."}
   ```
2. **IAM role (OIDC)** — one role, trust policy scoped to
   `repo:healfmx/*:*` (any healfmx repo, any branch), with a policy granting
   `secretsmanager:GetSecretValue` on the ARN above only.
3. **Per consuming repo**, add two repo secrets:
   - `PR_REVIEW_OIDC_ROLE_ARN` — the role ARN from step 2.
   - `PR_REVIEW_ANTHROPIC_SECRET_ARN` — the secret ARN from step 1.

## Adding PR review to a new repo

Add this file to the consuming repo as
`.github/workflows/pr-review.yml`:

```yaml
name: PR Review

on:
  pull_request:
    types: [opened] # deliberately not `synchronize` — see rationale below

jobs:
  review:
    uses: healfmx/healfmx-pr-agent-settings/.github/workflows/pr-review-reusable.yml@main
    secrets:
      OIDC_ROLE_ARN: ${{ secrets.PR_REVIEW_OIDC_ROLE_ARN }}
      ANTHROPIC_SECRET_ARN: ${{ secrets.PR_REVIEW_ANTHROPIC_SECRET_ARN }}
```

### Why `types: [opened]` only

PR-Agent's documented example workflow also triggers on `synchronize` (every
push to the PR). That re-runs the full command bundle per push, and cost
scales with commits-per-PR — measured on `healf-erp-backend` and
`healf-erp-customers`, that ranges from ~2 (median) to ~7 (mean, long-tail
PRs) re-runs per PR depending on how commits are grouped into pushes.
Restricting to `opened` makes cost per PR deterministic instead of a 6x
range. Add `synchronize` back deliberately, with the cost tradeoff understood,
if the team decides re-review-on-push is worth it.

### Why release PRs are excluded

The reusable workflow skips branches matching `release/*` — those PRs
package commits already reviewed individually against `develop`, so
re-running the bundle against the release diff adds a full model call
without new signal.

## Pilot / cost validation

Before rolling out to more repos, run this on one repo for a week with
`output_run_cost = true` (already set in `.pr_agent.toml`) and read the real
per-run cost PR-Agent posts, rather than relying on projected estimates.
