# healfmx-pr-agent-settings

Centralized PR-Agent configuration and reusable GitHub Actions workflow for
AI-powered automated Pull Request review (Claude Sonnet 5) across healfmx
repos.

## What lives here

- `.pr_agent.toml` — **reference copy** of the review config (model, commands
  run, cost tracking). Not fetched automatically by consuming repos — see
  "Config is per-repo, not centralized" below. Copy its contents into each
  consuming repo's own root `.pr_agent.toml` and keep them in sync by hand.
- `.github/workflows/pr-review-reusable.yml` — the reusable workflow every
  consuming repo calls. Takes the Anthropic API key as a plain GitHub secret
  and invokes PR-Agent. This part genuinely is shared/centralized and works
  as a reusable workflow.

## Config is per-repo, not centralized (important)

The original design pointed PR-Agent at this repo's `.pr_agent.toml` via a
`config.config_file` env var set to a `raw.githubusercontent.com` URL. **That
does not work.** Verified against a real run's debug log
(`healf-erp-backend` PR #277): the URL was stored as a literal config value,
never fetched, and PR-Agent silently ran with its built-in default model
(`gpt-5.6`, with a dummy OpenAI key) instead of anything in this repo's
`.pr_agent.toml`.

The only confirmed-working mechanism is a `.pr_agent.toml` at the
**consuming repo's own root** (PR-Agent reads that natively). So each
consuming repo needs the full config — not just repo-specific overrides —
copied from this repo's `.pr_agent.toml`, plus its own `repo_context_files`
list for its own AGENTS.md/CLAUDE.md layout.

## One-time setup (per consuming repo)

This account is an individual GitHub account, not an Organization, so there
is no org-wide secrets store — each consuming repo adds its own copy of the
same key.

Add one repo secret:

- `PR_REVIEW_ANTHROPIC_API_KEY` — the Anthropic API key, value as-is
  (`sk-ant-...`).

No AWS involved: this is a review-bot API key, not a production credential
(DB passwords, encryption keys, etc.), so it doesn't warrant the
Secrets-Manager-plus-OIDC pattern `healf-erp-backend` uses for those — a
plain GitHub secret is the right amount of ceremony here.

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
      ANTHROPIC_API_KEY: ${{ secrets.PR_REVIEW_ANTHROPIC_API_KEY }}
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

## Known limitation

Because config is per-repo (see above), a model/cost change here does not
propagate automatically — it has to be copied into every consuming repo's
`.pr_agent.toml` by hand. Revisit this if the number of consuming repos
grows enough that the manual sync becomes the actual maintenance burden.

## Pilot / cost validation

Before rolling out to more repos, run this on one repo for a week with
`output_run_cost = true` (already set in `.pr_agent.toml`) and read the real
per-run cost PR-Agent posts, rather than relying on projected estimates.
