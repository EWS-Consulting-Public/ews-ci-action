---
status: as-built
covers: ci.yml and release.yml - inputs, jobs, and the conditions that gate them
last-verified: 2026-08-07
---

# The two reusable workflows

Both are `workflow_call`-only. Neither runs on a push to this repository.

## `ci.yml`

Source: [`.github/workflows/ci.yml`](../.github/workflows/ci.yml).

### Inputs

| Input | Type | Default | Effect |
| --- | --- | --- | --- |
| `python-versions` | string | `'["3.12"]'` | JSON array; becomes the `test` job matrix |
| `lint-python-version` | string | `"3.12"` | Python for the `lint` and `build` jobs |
| `run-lint` | boolean | `true` | Run the `lint` job |
| `run-tests` | boolean | `true` | Run the `test` job |
| `run-coverage` | boolean | `true` | Run pytest with `--cov=src` and write `coverage.xml` |
| `upload-coverage` | boolean | `false` | Send `coverage.xml` to Codecov |
| `run-build` | boolean | `true` | Run the `build` job |
| `use-nox-build` | boolean | `true` | `uvx nox -s build` instead of `uv build --wheel` |
| `upload-artifacts` | boolean | `true` | Upload `dist/` for 7 days |
| `pytest-args` | string | `""` | Extra pytest arguments, e.g. `-m "not fails_in_ci"` |
| `ews-credentials-keys` | string | `""` | Forwarded to the composite action, which never reads it |

Secret: `EWS_CREDENTIALS`, **not required** by `ci.yml`. Omitting it is only
viable for a package with no private dependencies.

`run-coverage` is on and `upload-coverage` is off by default, so coverage is
measured every run and published on none unless the caller asks.

### Jobs

| Job | Runs when | Does |
| --- | --- | --- |
| `check-skip` | always | Inspects the head commit message and emits `should-skip` |
| `lint` | not skipped, `run-lint` | `uv run ruff check .` then `uv run ruff format --check .` |
| `test` | not skipped, `run-tests` | Matrix over `python-versions`; `uv run pytest` with or without coverage; `fail-fast: false` |
| `build` | not skipped, `run-build`, **after `lint` and `test`** | `uvx nox -s build` or `uv build --wheel`; uploads `dist/` |

`lint` calls the composite action with `install-dependencies: "false"` — ruff
is fetched by `uv run` on demand rather than syncing the whole project.

### The skip logic

`check-skip` sets `should-skip=true` in two cases (`ci.yml:78-97`):

1. The commit message contains `[skip ci]` or `[ci skip]`, case-insensitively.
2. The ref is `refs/heads/main` **and** the message's first line starts with
   `bump version to`. This is the version-bump case: the bump commit and the
   tag it creates would otherwise both trigger a full run, and the tag's run is
   the one that matters.

The message is read through the `COMMIT_MSG` environment variable rather than
interpolated into the script, so backticks and `$( )` in a commit message are
not executed (`ci.yml:74-77`). Keep it that way.

`github.event.head_commit.message` is empty on `pull_request` events, so
`[skip ci]` does not skip a PR.

## `release.yml`

Source: [`.github/workflows/release.yml`](../.github/workflows/release.yml).

### Inputs

| Input | Type | Default | Effect |
| --- | --- | --- | --- |
| `python-version` | string | `"3.12"` | Python for the build |
| `use-nox-publish` | boolean | `true` | `uvx nox -s publish` instead of an inline `twine upload` |
| `ews-credentials-keys` | string | `""` | Forwarded; never read |

Secret: `EWS_CREDENTIALS`, **required**.

### The trigger condition

The single `release` job runs only when both hold (`release.yml:35-37`):

- `github.event.workflow_run.conclusion == 'success'`
- `github.event.workflow_run.head_branch` starts with `v`

The caller must therefore drive it from a `workflow_run` event on the CI
workflow. `head_branch` carries the tag name for a tag-triggered run, which is
what the `v` prefix test is matching.

### Caller permissions

The workflow does not request permissions; the caller must
(`release.yml:26-27`):

```yaml
permissions:
  contents: write # create the GitHub release
  actions: read # read the triggering workflow run
```

### Steps

1. Checkout at `github.event.workflow_run.head_branch` — the tag.
2. `setup-ews-ci` with `install-dependencies: "false"`.
3. `uv lock` if no `uv.lock` is committed.
4. `uvx nox -s build` — unconditional. The consuming repository **must** have a
   `build` nox session even when it set `use-nox-build: false` in CI.
5. Publish: `uvx nox -s publish`, or an inline `twine upload` of `dist/*.whl`
   that re-reads the two GitLab values out of `EWS_CREDENTIALS` with `jq`.
6. `softprops/action-gh-release@v1` with `dist/*` and `uv.lock` attached and
   `generate_release_notes: true`.
7. A job summary listing the built files and an install command, with the
   package name scraped out of `pyproject.toml` by `grep`/`sed`
   (`release.yml:94`) — this assumes a top-level `name = "..."` line and will
   pick up the first match, so a `[project]` name must appear before any other
   `name = ` key.

The wheel is **rebuilt** here, not downloaded from CI
([ADR 0005](decisions/0005-release-rebuilds-rather-than-downloading.md)).

## Not verified

No run of either workflow was observed. Job ordering, conditions and commands
are read from the YAML.
