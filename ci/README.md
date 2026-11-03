# `ci`: how the pipeline builds, documents and publishes itself

Everything automated in this project lives in one GitHub Actions workflow, [`.github/workflows/dbt_build.yml`](../.github/workflows/dbt_build.yml). This folder holds the dbt profile it builds with, [`profiles.yml`](profiles.yml).

The workflow does three jobs. Which ones run depends on what started it:

| Trigger | `dbt build` (models + tests) | Regenerate docs in `docs/` | Publish `prod.duckdb` |
|---|:-:|:-:|:-:|
| Pull request into `main` | ✅ | ✅ | ❌ |
| Push to `main` (e.g. merging a PR) | ✅ | ❌ | ✅ |
| Monday schedule (06:17 UTC) | ✅ | ❌ | ✅ |
| Manual run (**Run workflow** button) | ✅ | ❌ | ✅ |

**Why docs only on pull requests?** The docs describe the project's code: models, columns, tests and lineage. That code only changes through a PR, so a PR is the only time the docs can go stale. The workflow regenerates them on the PR branch, so the PR carries code and docs together. When you merge, `main` (and the GitHub Pages site built from `docs/`) gets both at once. Normal builds only refresh the data, so they skip the docs and don't add noise commits.

**Why no publishing on pull requests?** The PR's code isn't merged yet. If a PR build uploaded `prod.duckdb`, the public warehouse would show unreviewed changes.

---

## The workflow, step by step

### Triggers

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '17 6 * * 1'
  workflow_dispatch:
```

- `push` to `main` runs whenever code lands on `main`. In practice, that's when a PR is merged.
- `pull_request` into `main` runs when a PR is opened or reopened, and on every new commit pushed to it.
- `schedule` rebuilds every Monday at 06:17 UTC so the warehouse picks up fresh data without anyone pushing. The odd minute avoids the top-of-the-hour rush on GitHub's runners.
- `workflow_dispatch` adds a manual **Run workflow** button in the Actions tab.

### Permissions

```yaml
permissions:
  contents: write
```

By default, the workflow's `GITHUB_TOKEN` is read-only. `contents: write` lets it do two things: upload `prod.duckdb` to the `warehouse` release, and push the regenerated docs commit to a PR branch.

### Concurrency

```yaml
concurrency:
  group: dbt-build-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

Runs are grouped by ref: each PR gets its own group, and `main` has one group. Only one run per group happens at a time.

- **On a PR**, a new push cancels the run still in progress. Only the latest commit matters, and two runs pushing docs to the same branch would collide.
- **On `main`**, runs queue instead of cancelling, so a scheduled build never kills a publish halfway through.

### Checkout

```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.head_ref || github.ref }}
```

On a pull request, GitHub normally checks out a temporary *merge commit*. You can't push that back anywhere. `github.head_ref` is the PR's own branch name, so the job checks out the real branch and can commit to it. On every other trigger `head_ref` is empty, and the expression falls back to `github.ref` (`main`).

### Install, download, build

```yaml
- uses: astral-sh/setup-uv@v5
- run: uv sync
- run: uv run python scripts/download_data.py
- run: uv run dbt build --profiles-dir ../ci
  working-directory: analytics
```

1. `setup-uv` installs [uv](https://docs.astral.sh/uv/), the Python package manager.
2. `uv sync` installs the exact dependency versions pinned in `uv.lock`, dbt among them.
3. `download_data.py` fetches the raw parquet files into `data/raw/`.
4. `dbt build` runs every model and test in DAG order. `--profiles-dir ../ci` tells dbt to use [`ci/profiles.yml`](profiles.yml) instead of `~/.dbt/profiles.yml`, which doesn't exist on the runner. That profile's `prod` target writes to `data/prod.duckdb`.

If any test fails, the job stops here. Nothing gets documented or published.

### Generate the docs (pull requests only)

```yaml
- name: Generate dbt docs into docs/
  if: github.event_name == 'pull_request'
  run: |
    find ../docs -mindepth 1 ! -name .nojekyll -exec rm -rf {} +
    uv run dbt docs generate --no-compile --profiles-dir ../ci --output-dir ../docs
  working-directory: analytics
```

- **The `find … rm` line empties `docs/` but keeps `.nojekyll`.** `dbt docs generate` adds files but never removes old ones. The site's JavaScript files have content hashes in their names (`index-DcNiPm6H.js`), so without this cleanup every regeneration would leave outdated files behind. `.nojekyll` stays because it tells GitHub Pages to serve the folder as-is.
- **`--output-dir ../docs`** writes the static site (`index.html`, `assets/`, and the parquet metadata in `info_schema/`) straight into the folder GitHub Pages serves.
- **`--no-compile`** reuses the project index the `dbt build` step just wrote, so the project isn't compiled a second time.

### Commit the docs to the PR branch (pull requests only)

```yaml
- name: Commit updated docs to the PR branch
  if: >-
    github.event_name == 'pull_request' &&
    github.event.pull_request.head.repo.full_name == github.repository
  env:
    HEAD_REF: ${{ github.head_ref }}
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
    git add docs
    if git diff --cached --quiet; then
      echo "Docs unchanged"
    else
      git commit -m "docs: regenerate dbt docs"
      git pull --rebase origin "$HEAD_REF"
      git push origin "HEAD:$HEAD_REF"
    fi
```

- **The `if` condition** also requires the PR branch to belong to this repository. PRs from forks get a read-only token and couldn't push anyway, so they skip this step. Their docs are still generated and checked.
- **`HEAD_REF` goes in through `env`** rather than being pasted into the script as `${{ github.head_ref }}`. Branch names are chosen by whoever opens the PR. Passing the name as an environment variable means it's always treated as text, never run as shell code.
- **The bot identity** (`github-actions[bot]`) makes these commits easy to tell apart from yours in the history.
- **`git diff --cached --quiet`** skips the commit when nothing changed.
- **`git pull --rebase`** before pushing covers the case where you pushed to the PR branch while the job was running.
- **This push doesn't start another run.** GitHub never triggers workflows from pushes made with `GITHUB_TOKEN`, which prevents an endless loop.

> 💡 After CI adds its docs commit, run `git pull` on your PR branch before pushing again.

### Publish the warehouse (everything except pull requests)

```yaml
- name: Publish prod.duckdb to the warehouse release
  if: github.event_name != 'pull_request'
  run: |
    gh release create warehouse --notes "Warehouse — auto-built by dbt" || true
    gh release upload warehouse data/prod.duckdb --clobber
  env:
    GH_TOKEN: ${{ github.token }}
```

- `gh release create … || true` creates the `warehouse` release the first time. On later runs the release already exists, and `|| true` stops that "already exists" error from failing the job.
- `gh release upload … --clobber` replaces the previous `prod.duckdb` with the new one. The download URL in the main README never changes.

---

## Things to know

- **The docs commit has no checks of its own.** The bot's push doesn't trigger a workflow, so GitHub shows no CI status on that commit. If you add branch protection on `main` that requires status checks, the PR will look like it's waiting. You'd then need a personal access token or GitHub App token to push the docs commit.
- **A PR touching only data won't change much in the docs.** The site also stores a `generated_at` timestamp, though, so almost every PR run still makes a small docs commit.
- **Build docs locally anytime** with `uv run dbt docs generate` from `analytics/`. It writes to `analytics/target/` by default, not `docs/`, so local runs won't touch the published site.

← Back to the [project README](../README.md).
