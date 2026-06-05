# readme-updater

A drop-in GitHub Actions workflow that keeps a repo's `README.md` current by
having a cheap LLM (Gemini Flash) read the **last day of commits** once a day and
edit the README where it judges the changes matter. It commits the result
directly — no human in the loop.

The workflow lives at
[`.github/workflows/readme-auto-update.yml`](.github/workflows/readme-auto-update.yml).
Copy that single file into any repo you want auto-documented.

## Setup (per repo)

1. **Get a free Gemini key** at <https://aistudio.google.com/apikey> → *Create API key*.
2. **Add the secret:** repo → *Settings → Secrets and variables → Actions → New
   repository secret* → name it `GEMINI_API_KEY`.
3. **Copy the workflow file** into the target repo at the same path and commit it.
4. **Test it now:** *Actions* tab → *README auto-update (AI)* → *Run workflow*.

## How it behaves

- **Skips cleanly** when there were zero commits in the window (no model call, no
  empty commit).
- **Only commits when the README actually changed** (`git-auto-commit-action`
  no-ops otherwise).
- **`temperature: 0`** plus an explicit "return unchanged if nothing warrants it"
  instruction keeps diffs from being noisy reflows.
- **No infinite loop:** the push uses the default `GITHUB_TOKEN` (which doesn't
  re-trigger workflows) and the commit carries `[skip ci]`.

## Knobs

| What | Where | Notes |
| --- | --- | --- |
| Model | `GEMINI_MODEL` env | `gemini-2.5-flash-lite` is cheaper than the default `gemini-2.5-flash`. |
| Cadence | `cron` | Weekly: `'0 6 * * 1'`. Cron is UTC. |
| Commit window | `SINCE` env | Widen to `'2 days ago'` for overlap insurance if a scheduled run is skipped. |
| Diff context size | `head -c 24000` | Larger = more context, slightly higher cost. |

## Gotchas

- **GitHub pauses scheduled workflows after 60 days of no repo activity** — a
  dormant repo stops firing until you push or run it manually.
- The 24h window assumes the daily cron actually runs; GitHub can delay scheduled
  jobs under load. Widen `SINCE` if you want overlap insurance.

## Want a PR instead of a direct commit?

Swap the final step for `peter-evans/create-pull-request@v6` and add
`pull-requests: write` to `permissions`.
