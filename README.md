# readme-updater

A GitHub Action that keeps a repo's `README.md` current. On a schedule, a **Claude
agent investigates what actually changed** — reading commits, merged PR
descriptions, referenced issues, and the touched source files via `git` and the
`gh` CLI — and then edits the README only where it materially matters. It commits
the result **only if `README.md` is the sole changed file**; if the agent touched
anything else, the run reverts and fails instead of pushing.

It's a **composite action** — drop ~13 lines into any repo and point at `@v2`.

> **Two tiers.** `@v2` (this) is the **agentic investigator** (Anthropic key, more
> capable, costs cents per run). `@v1` is a cheap one-shot that just feeds a
> truncated diff to **Gemini Flash** (sub-cent, no investigation). Pick per repo —
> see [Legacy: the cheap `@v1` one-shot](#legacy-the-cheap-v1-one-shot).

## Install (per repo)

1. **Get an Anthropic API key** at <https://console.anthropic.com> → *API keys*.
2. **Add the secret:** target repo → *Settings → Secrets and variables → Actions →
   New repository secret* → name it `ANTHROPIC_API_KEY`. (Or set it once as an
   **organization secret** shared to selected repos.)
3. **Add this workflow** at `.github/workflows/readme.yml`:

   ```yaml
   name: README auto-update (AI)
   on:
     schedule:
       - cron: '0 6 * * *'      # daily 06:00 UTC
     workflow_dispatch: {}
   permissions:
     contents: write            # so the action can push the README change
     pull-requests: read        # so the agent can crawl merged PRs via gh
     issues: read               # so the agent can crawl referenced issues via gh
   jobs:
     update-readme:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
           with:
             fetch-depth: 0      # the agent needs history to investigate the window
         - uses: prbe-ai/readme-updater@v2
           with:
             anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
   ```

4. **Test it:** *Actions* tab → *README auto-update (AI)* → *Run workflow*.

> The API key is **never** stored in the repo. `${{ secrets.ANTHROPIC_API_KEY }}`
> is only a reference; the value lives in GitHub's encrypted secret store, separate
> from your code — safe in public repos, and never shared with forks.

## What it does

1. Records the base commit.
2. Runs a Claude agent (`anthropics/claude-code-action`, pinned to a full SHA) with a
   **read-only** toolset: `Read`, `Edit`, and read subcommands of `git` and `gh`
   (`gh pr list/view/diff`, `gh issue list/view`, `gh search`, `gh api`). It is **not**
   allowed to commit, push, or open PRs.
3. The agent investigates the window, confirms behavior in the source, and edits the
   README.
4. A guard step commits **only if `README.md` is the single changed path** (checking
   both uncommitted edits and any commits the agent made). Anything else → hard
   revert + fail.

## Inputs

| Input | Required | Default | Notes |
| --- | --- | --- | --- |
| `anthropic-api-key` | yes | — | Pass via `with:` from a secret. Never hardcode. |
| `model` | no | `claude-haiku-4-5` | Cheap by default; bump for harder repos. |
| `since` | no | `1 day ago` | Investigation window. Match it to your cron. |
| `readme-path` | no | `README.md` | The **only** file the agent may change. |
| `max-turns` | no | `20` | Caps agent tool-use turns (cost guard). |
| `commit` | no | `true` | `false` leaves the edit in the working tree (e.g. to open a PR yourself). |
| `commit-message` | no | `docs: auto-update README from recent commits [skip ci]` | Used when committing. |

## Safety & cost

- **README-only guarantee.** The agent can only ever land a change to `readme-path`.
  Source/workflow edits are reverted and the run fails loudly.
- **Least privilege.** The example grants `contents: write` (to push) plus read-only
  `pull-requests`/`issues` (so `gh` can crawl). With no write scope on PRs/issues, a
  misbehaving `gh api` POST is rejected by the token.
- **Pinned action.** `claude-code-action` is pinned to a full commit SHA per its
  [security advisory](https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/)
  (an agent reading commit messages is reading attacker-influenceable text).
- **Cost.** This is an agent loop, not one call — expect cents per run on Haiku.
  `max-turns` bounds it. For sub-cent runs with no investigation, use `@v1`.
- **No loop:** the push uses the default `GITHUB_TOKEN` (doesn't re-trigger
  workflows) and the commit carries `[skip ci]`.

## Want a PR instead of a direct commit?

Set `commit: false`, then add a `peter-evans/create-pull-request@v6` step after the
action and `pull-requests: write` to `permissions`. The action leaves the edited
README in the working tree for the PR step.

## Gotchas

- **GitHub pauses scheduled workflows after 60 days of no repo activity** — a
  dormant repo stops firing until you push or run it manually.
- The window assumes the cron actually runs; GitHub can delay scheduled jobs under
  load. Widen `since` (e.g. `'2 days ago'`) for overlap insurance.
- The consumer workflow **must** check out with `fetch-depth: 0`.

## Legacy: the cheap `@v1` one-shot

`@v1` predates the agent. It feeds a truncated `git log -p` straight to **Gemini
Flash** in a single call — no investigation, sub-cent, uses `GEMINI_API_KEY`:

```yaml
- uses: actions/checkout@v4
  with: { fetch-depth: 0 }
- uses: prbe-ai/readme-updater@v1
  with:
    gemini-api-key: ${{ secrets.GEMINI_API_KEY }}
```

Use `@v1` for cheap mechanical refreshes, `@v2` when you want the README to reflect
what genuinely changed. Pin `@v2`/`@v1` for the moving major tag, or
`@v2.0.0`/`@v1.0.0` to lock exactly.

## Requirements

`git` and `gh` (preinstalled on GitHub-hosted `ubuntu-latest` runners). `@v1` also
uses `jq`/`curl` (also preinstalled). No other dependencies.

---

This repo dogfoods its own action — see
[`.github/workflows/readme-auto-update.yml`](.github/workflows/readme-auto-update.yml).
