# AGENTS.md

Guidance for coding agents working in this repository. Renamed from `CLAUDE.md` to the cross-agent
`AGENTS.md` convention on 2026-10-21.

## What this repo is

This is the GitHub **profile README** repo (`Talyra42/Talyra42`). Its only user-facing artifact is [README.md](README.md), which renders on the profile page. There is no application, build, lint, or test suite. The "code" here is three GitHub Actions workflows that generate SVG charts; everything else is Markdown.

## The output-branch model (most important concept)

All generated charts live on a separate orphan-style branch named **`output`**, *not* on `main`. README references them by raw URL, e.g. `https://raw.githubusercontent.com/Talyra42/Talyra42/output/<file>.svg`. This makes the README depend only on this repo's own files — never on a third-party live rendering service (a previous third-party stats host went 404 and broke the README).

Three workflows write to `output`, so they must not clobber each other:

- [.github/workflows/snake.yml](.github/workflows/snake.yml) — `Platane/snk` contribution-snake animation, published to `output` via `crazy-max/ghaction-github-pages`. That action does a *clean publish*, so it is configured with **`keep_files: true`** to preserve the metrics SVGs. Runs at `0 0 * * *`.
- [.github/workflows/metrics.yml](.github/workflows/metrics.yml) — `lowlighter/metrics` renders `metrics.svg` and `metrics-languages.svg`, committed to `output` (`output_action: commit`, `committer_branch: output`). Scheduled at `30 */6 * * *` — deliberately **offset off the hour** so it never pushes to `output` at the same instant as the snake workflow (which would cause a non-fast-forward push conflict). Has no `push:` trigger for the same reason.
- [.github/workflows/daily-commit.yml](.github/workflows/daily-commit.yml) — writes to `main`, not `output`; see below.

`lowlighter/metrics` requires `committer_branch` to **already exist** — it reads that branch's SHA to base its commit on and does not create it. Keep generated charts on the existing `output` branch.

## Workflow gotchas (learned the hard way)

- **Languages card needs indepth mode.** `lowlighter/metrics`' default languages mode reads `data.user.repositories.nodes[].languages.edges`, which comes back empty for this account, producing a blank card. The fix in use is `plugin_languages_indepth: yes` (clones repos and runs Linguist on real file contents). Do not revert to default mode.
- **Daily contribution commit must be authored by a verified account email.** GitHub's contribution graph attributes by the *commit author email*, not the pusher. Bot-authored commits (`github-actions[bot]`) from the snake/metrics workflows do **not** count. `daily-commit.yml` sets `git config user.email` to a GitHub-account-verified email so its empty commits count. Runs at `0 4 * * *` (UTC) = 12:00 Asia/Shanghai.
- **A dead `METRICS_TOKEN` fails the job — it does not degrade gracefully.** Both metrics steps use `token: ${{ secrets.METRICS_TOKEN || github.token }}`, but that `||` only falls back when the secret is *missing or empty*. A secret that still exists but holds an expired or revoked token is passed through verbatim, every API call returns `401 Bad credentials`, and `lowlighter/metrics` retries 3× for ~15 minutes before failing. The only visible symptom is that the charts silently stop updating. This is what happened on 2026-10-21: the token stored on 2026-06-06 had been deleted, and run `39597314208` burned 15m48s before failing. `metrics.yml` now starts with a `Preflight - verify METRICS_TOKEN` step that turns this into a ~10 second failure with instructions.
- **Classic PAT only, scopes `repo` + `read:user`.** `lowlighter/metrics@v3.34` throws immediately on fine-grained tokens (`github_pat_…`) because GraphQL does not accept them. `repo` is what enables the private-contribution stats; without it metrics still renders, on public data only.
- **Rotating the token.** Create a classic PAT → `gh secret set METRICS_TOKEN -R Talyra42/Talyra42` (paste at the hidden prompt; never commit it) → `gh workflow run metrics.yml` → confirm `Token validity │ seems ok` in the log and a fresh commit on `output`. After rotating, replace the placeholder on the next line with the date the new token expires, so it can be rotated ahead of time instead of after CI breaks.

```
METRICS_TOKEN expires: <YYYY-MM-DD>
```

## How to validate changes (there are no tests)

Workflows only run on GitHub, so the dev loop is: edit → push → trigger → inspect the `output` branch.

```bash
# Validate a workflow's YAML before pushing (when python3 is available)
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/metrics.yml'))"

# Manually trigger a workflow and watch it.
# A 422 from `gh workflow run` also means the YAML is invalid.
gh workflow run metrics.yml
gh run list --workflow=metrics.yml --limit 1
gh run view <run-id> --log         # full log of a run

# Inspect a generated SVG without CDN caching (raw.githubusercontent caches ~5 min)
git fetch origin output && git show FETCH_HEAD:metrics-languages.svg
```

When editing the README's analytics section, point `<img>` tags at `.../<branch>/<file>.svg` on the `output` branch, not at any external service.
