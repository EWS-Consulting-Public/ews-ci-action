---
status: as-built
covers: ci.yml, release.yml and e2e.yml - inputs, jobs, the data plan, and the conditions that gate them
last-verified: 2026-10-07
---

# The reusable workflows

All three are `workflow_call`-only. None runs on a push to this repository,
except that `e2e-selftest.yml` calls `e2e.yml` on this repository's own pull
requests (§ *The self-test*).

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
| `ews-credentials-keys` | string | `""` | Forwarded to the composite action, which fails the job if `EWS_CREDENTIALS` lacks a key it names |
| `data-plan` | string | `""` | Path of a data plan in the checkout. Non-empty: the `test` job caches the data root on the plan's lock and runs `ews-storage plan ensure` before pytest — § *The data plan* |

Secret: `EWS_CREDENTIALS`, **not required** by `ci.yml`. Omitting it is only
viable for a package with no private dependencies.

`run-coverage` is on and `upload-coverage` is off by default, so coverage is
measured every run and published on none unless the caller asks.

### Jobs

| Job | Runs when | Does |
| --- | --- | --- |
| `check-skip` | always | Inspects the head commit message and emits `should-skip` |
| `lint` | not skipped, `run-lint` | `uv run ruff check .` then `uv run ruff format --check .` |
| `test` | not skipped, `run-tests` | Matrix over `python-versions`; the data plan steps when `data-plan` is set; `uv run pytest` with or without coverage; `fail-fast: false` |
| `build` | not skipped, `run-build`, **after `lint` and `test`** | `uvx nox -s build` or `uv build --wheel`; uploads `dist/` |

`lint` calls the composite action with `install-dependencies: "false"` — ruff
is fetched by `uv run` on demand rather than syncing the whole project.

### The data plan

A package whose tests read a dataset held in the bucket names the plan that
selects what those tests need:

```yaml
with:
  data-plan: config/plans/fixtures.yaml
```

The `test` job then runs four steps between the composite action and pytest,
each skipped entirely when the input is empty:

1. **Resolve** — the plan must exist, and so must its lock,
   `<plan without extension>.lock.json`, beside it. The cache key is
   `ews-data-<OS>-<first 16 hex of sha256(lock)>`. A missing lock fails the
   job with the command that writes it.
2. **Restore** `actions/cache/restore@v4` on the data root the composite
   action exported (`$GITHUB_WORKSPACE/.ews-data`), under that key, with
   `ews-data-<OS>-` as the prefix fallback.
3. **Fetch** `uv run ews-storage plan ensure <plan>`, **unconditionally**. On an
   exact hit it verifies and fetches nothing; after a prefix restore it fetches
   only what differs; on a cold runner it fetches everything. The step's log
   ends with one of three lines, built from the restore step's `cache-hit` and
   `cache-matched-key` outputs, and the same information goes to the job
   summary as a table.
   - Exact hit: `plan-ensure took <n>s after an exact cache hit on <key>`.
   - Prefix restore: `cache-hit` was not `true`, but `cache-matched-key` named
     an older key under the `ews-data-<OS>-` prefix, so the log reads
     `plan-ensure took <n>s after a cache miss on <key>: restored <matched
     key> instead, an older cache under the restore-keys prefix, so the data
     root was already populated, and a 0-object fetch only proves the
     digests still match, not that the store was reached`. This is the case
     a plain hit or miss reading gets wrong: the exact key missed, but the
     data root was not empty, and a run that fetched nothing did not prove
     the object store was reachable.
   - Cold: neither output was set, so `plan-ensure took <n>s after a cache
     miss on <key>, nothing restored`.
4. **Save** `actions/cache/save@v4` under the same key, only when the restore
   was not an exact hit — and before pytest, so a red test still leaves a warm
   cache for the next run.

What the consuming repository must have: `ews-cloud-storage` among its
dependencies (it provides `ews-storage`, and the composite action's `uv sync`
has already installed it); in its `EWS_CREDENTIALS`, the keys of the store the
dataset document points at, by its scheme (table below); a committed lock
beside the plan, kept current by that package's own hook; and a dataset
document that binds its data root to `EWS_DATA_ROOT`: a document bound to
another variable lands its files outside the cached directory. Why the cache
is the whole data root and the key is the plan's lock:
[ADR 0006](decisions/0006-data-plan-caches-the-data-root-on-the-plan-lock.md).

| Dataset store | Keys in `EWS_CREDENTIALS` | Exported as, and read by `plan ensure` |
| --- | --- | --- |
| `gs://`, Google Cloud Storage | `gcp_default_key` | `GCP_DEFAULT_KEY` |
| `s3://`, the S3-compatible store | `synologyc2_default_key_id`, `synologyc2_default_secret_key` | `SYNOLOGYC2_DEFAULT_KEY_ID`, `SYNOLOGYC2_DEFAULT_SECRET_KEY` |

The composite action does not know the scheme. It exports every key the secret
carries, so a plan on either store needs nothing here beyond `data-plan`.

The lint and build jobs do not run the data steps; only tests read data.

### The skip logic

`check-skip` sets `should-skip=true` in two cases:

1. The commit message contains `[skip ci]` or `[ci skip]`, case-insensitively.
2. The ref is `refs/heads/main` **and** the message's first line starts with
   `bump version to`. This is the version-bump case: the bump commit and the
   tag it creates would otherwise both trigger a full run, and the tag's run is
   the one that matters.

The message is read through the `COMMIT_MSG` environment variable rather than
interpolated into the script, so backticks and `$( )` in a commit message are
not executed. Keep it that way; the data plan path takes the same route
(`DATA_PLAN` in `env:`).

`github.event.head_commit.message` is empty on `pull_request` events, so
`[skip ci]` does not skip a PR.

## `release.yml`

Source: [`.github/workflows/release.yml`](../.github/workflows/release.yml).

### Inputs

| Input | Type | Default | Effect |
| --- | --- | --- | --- |
| `python-version` | string | `"3.12"` | Python for the build |
| `use-nox-publish` | boolean | `true` | `uvx nox -s publish` instead of an inline `twine upload` |
| `ews-credentials-keys` | string | `""` | Forwarded to the composite action, which validates the secret against it |

Secret: `EWS_CREDENTIALS`, **required**.

### The trigger condition

The single `release` job runs only when both hold:

- `github.event.workflow_run.conclusion == 'success'`
- `github.event.workflow_run.head_branch` starts with `v`

The caller must therefore drive it from a `workflow_run` event on the CI
workflow. `head_branch` carries the tag name for a tag-triggered run, which is
what the `v` prefix test is matching.

### Caller permissions

The workflow does not request permissions; the caller must:

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
   package name scraped out of `pyproject.toml` by `grep`/`sed` — this assumes
   a top-level `name = "..."` line and will pick up the first match, so a
   `[project]` name must appear before any other `name = ` key.

The wheel is **rebuilt** here, not downloaded from CI
([ADR 0005](decisions/0005-release-rebuilds-rather-than-downloading.md)).

## `e2e.yml`

Source: [`.github/workflows/e2e.yml`](../.github/workflows/e2e.yml). Copy-paste
caller: [`examples/e2e.yml`](../examples/e2e.yml).

One command, run on a checkout, with the declared secrets in its environment.
The workflow has no trigger of its own: the consumer's caller owns the `on:`
block, so the consumer decides when it runs. Why it is a separate workflow and
why it declares secrets by name:
[ADR 0007](decisions/0007-e2e-workflow-the-consumer-triggers.md).

### Inputs

| Input | Type | Default | Effect |
| --- | --- | --- | --- |
| `command` | string | **required** | The command to run. It reaches the shell through `env:` only |
| `ref` | string | `""` | The git ref to check out. Empty is the ref that triggered the caller |
| `python-version` | string | `"3.12"` | Python for the run |
| `ews-credentials-keys` | string | `""` | Forwarded to the composite action, which validates the secret against it |

### Secrets

None is required.

| Secret | Reaches |
| --- | --- |
| `EWS_CREDENTIALS` | The setup step. Without it the setup step installs nothing private |
| `EWS_GCP__DEFAULT_KEY` | The `End-to-end` step only, as an environment overlay of the EWS configuration layer |
| `EWS_GCP__PROJECT` | The `End-to-end` step only, as an environment overlay of the EWS configuration layer |

### Steps

The single `e2e` job runs on `ubuntu-latest`:

1. Checkout at `ref`.
2. `setup-ews-ci` with `install-dependencies: "false"`, as `release.yml` calls it.
3. `End-to-end`, with no `if:` because the command is required. The command and
   the two overlay secrets reach the step through `env:`. The step logs
   `<NAME>: set` or `<NAME>: not passed` for each secret, unsets one that is
   empty (an exported empty overlay would be read as a value, not as absent),
   then runs `bash -euo pipefail -c "$E2E_COMMAND"`.

The command is caller-controlled. Keep it in `env:` and never interpolate it
into a `run:` block. The same holds for the commit message a caller tests in its
`if:`: it is read in that expression and nowhere else.

### The triggers

[`examples/e2e.yml`](../examples/e2e.yml) shows three: a manual
`workflow_dispatch`, a push to `main` whose commit message contains `[e2e]`, and
the completion of the caller's release workflow. A caller keeps the ones it wants.
The after-release line also tests that the run was for a `v` tag, because a
release workflow without a `branches: ['v*']` filter runs after every CI run
and only its job is skipped.

GitHub limits a `workflow_run` chain to three levels. The after-release trigger
is the third (CI, then the release, then this), and that path is not exercised
here.

### The self-test

[`.github/workflows/e2e-selftest.yml`](../.github/workflows/e2e-selftest.yml)
calls `e2e.yml` on this repository, on a manual dispatch and on a pull request
that touches either file. It passes `EWS_GCP__PROJECT` as a literal dummy and
leaves `EWS_GCP__DEFAULT_KEY` out. The command succeeds only if the first is set
and the second is unset (`${VAR+set}` tells unset from empty). It needs no
repository secret.

## Not verified

Job ordering, conditions and commands are read from the YAML. The data plan
steps were first observed on a runner on 2026-09-05 in a purpose-built
consuming repository; `release.yml` has not been observed here, and neither has `e2e.yml`,
including its self-test, which runs only once pushed.
