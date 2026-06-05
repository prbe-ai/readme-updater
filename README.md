# readme-updater

A GitHub Action that keeps a repo's `README.md` current. Once a day (or on any
schedule you pick) it digests recent commits with a cheap LLM (Gemini Flash) and
edits the README only where the changes actually warrant it, then commits the
result. No human in the loop.

It's a **composite action** — drop ~12 lines into any repo and point at it.

## Install (per repo)

1. **Get a Gemini key** at <https://aistudio.google.com/apikey> → *Create API key*.
2. **Add the secret:** target repo → *Settings → Secrets and variables → Actions →
   New repository secret* → name it `GEMINI_API_KEY`. (Or set it once as an
   **organization secret** and share it across repos.)
3. **Add this workflow** at `.github/workflows/readme.yml` in the target repo:

   ```yaml
   name: README auto-update (AI)
   on:
     schedule:
       - cron: '0 6 * * *'      # daily 06:00 UTC
     workflow_dispatch: {}
   permissions:
     contents: write            # so the action can push the README change
   jobs:
     update-readme:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
           with:
             fetch-depth: 0      # the action needs history to diff the commit window
         - uses: prbe-ai/readme-updater@v1
           with:
             gemini-api-key: ${{ secrets.GEMINI_API_KEY }}
   ```

4. **Test it:** *Actions* tab → *README auto-update (AI)* → *Run workflow*. (On a
   repo with no commits in the window it just no-ops.)

> The API key is **never** stored in the repo. `${{ secrets.GEMINI_API_KEY }}` is
> only a reference; the value lives in GitHub's encrypted secret store, separate
> from your code — safe in public repos, and never shared with forks.

## Inputs

| Input | Required | Default | Notes |
| --- | --- | --- | --- |
| `gemini-api-key` | yes | — | Pass via `with:` from a secret. Never hardcode. |
| `gemini-model` | no | `gemini-2.5-flash` | `gemini-2.5-flash-lite` is cheaper. |
| `since` | no | `1 day ago` | Commit window for `git log --since`. Match it to your cron. |
| `readme-path` | no | `README.md` | File to maintain. |
| `diff-budget` | no | `24000` | Max bytes of diff sent to the model (cost guard). |
| `commit` | no | `true` | `false` leaves the edit in the working tree (e.g. to open a PR yourself). |
| `commit-message` | no | `docs: auto-update README from recent commits [skip ci]` | Used when `commit: true`. |

The action also sets a `readme-changed` step output (`true`/`false`).

## How it behaves

- **Skips cleanly** when there were zero commits in the window — no model call, no
  empty commit.
- **Only commits when the README actually changed.**
- **`temperature: 0`** plus a strict "return unchanged if nothing warrants it"
  instruction keeps diffs from being noisy reflows.
- **No infinite loop:** the push uses the default `GITHUB_TOKEN` (which doesn't
  re-trigger workflows) and the commit carries `[skip ci]`.

## Want a PR instead of a direct commit?

Set `commit: false`, then add a `peter-evans/create-pull-request@v6` step after the
action and `pull-requests: write` to `permissions`. The action leaves the edited
README in the working tree for the PR step to pick up.

## Gotchas

- **GitHub pauses scheduled workflows after 60 days of no repo activity** — a
  dormant repo stops firing until you push or run it manually.
- The window assumes the cron actually runs; GitHub can delay scheduled jobs under
  load. Widen `since` (e.g. `'2 days ago'`) if you want overlap insurance.
- The consumer workflow **must** check out with `fetch-depth: 0` and grant
  `permissions: contents: write`.

## Requirements

`jq` and `curl` (preinstalled on GitHub-hosted `ubuntu-latest` runners). No other
dependencies.

---

This repo dogfoods its own action — see
[`.github/workflows/readme-auto-update.yml`](.github/workflows/readme-auto-update.yml).
