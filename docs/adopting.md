---
status: as-built
covers: putting these workflows into a consuming repository
last-verified: 2026-08-07
---

# Adopting `ews-ci-action`

## Before you start

The consuming repository needs:

- `pyproject.toml`, resolvable by `uv`. A committed `uv.lock` is used with
  `--frozen`; without one the action locks on the fly.
- `ruff` and `pytest` runnable through `uv run` (default `run-lint` /
  `run-tests`).
- A `nox` session named `build` (default `use-nox-build: true`). Set
  `use-nox-build: false` to use `uv build --wheel` instead — but note that
  `release.yml` calls `nox -s build` unconditionally, so a repository that
  releases needs the session regardless.
- A `nox` session named `publish` if it releases with the default
  `use-nox-publish: true`.
- An `EWS_CREDENTIALS` secret — see [credentials.md](credentials.md).

## 1. CI

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@v1
    with:
      python-versions: '["3.12", "3.13"]'
    secrets:
      EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }}
```

Include the `tags: ["v*"]` trigger. The release workflow keys off a *successful
CI run on a tag*, so a CI that ignores tags never produces a release.

## 2. Release

`.github/workflows/release.yml`:

```yaml
name: Release

on:
  workflow_run:
    workflows: ["CI"] # must match the CI workflow's `name:`
    types: [completed]

permissions:
  contents: write
  actions: read

jobs:
  release:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/release.yml@v1
    secrets:
      EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }}
```

Three things here are load-bearing:

- **`workflows: ["CI"]` matches the CI workflow's `name:` field**, not its
  filename.
- **The `permissions` block is required.** The reusable workflow does not
  request permissions for you, and creating a GitHub release needs
  `contents: write`.
- **Do not filter this trigger on `branches`.** Tag runs are not branch runs;
  the reusable workflow already tests `head_branch` for the `v` prefix itself.

## 3. Release flow

1. Tag a version and push the tag (`v20260807.0`, or whatever scheme the
   package uses).
2. CI runs on the tag. If it goes green, the `workflow_run` event fires.
3. `release.yml` checks out the tag, builds, publishes to the registry, and
   creates a GitHub release with the wheel and `uv.lock` attached.

If the version bump is a commit on `main` whose message starts with
`bump version to`, `check-skip` suppresses the duplicate CI run for it; the
tag's run is the one that matters.

## Examples

[`examples/`](../examples/) holds three copy-paste callers:
`basic-package.yml`, `matrix-testing.yml`, `release-basic.yml`.

They use `secrets: inherit`, which passes every secret the caller repository
has. That is convenient and coarse; naming `EWS_CREDENTIALS` explicitly, as
above, is the narrower form. `matrix-testing.yml` passes no secrets at all — it
only works for a package with no private dependencies.

Two of them are behind this page. Copy the snippets above rather than the files:

- **`release-basic.yml` filters `workflow_run` on `branches: ['v*']`.** That is
  the deprecated pattern § 2 tells you not to use: tag runs are not branch runs,
  so the filter can stop the release from ever firing, and the reusable workflow
  already tests `head_branch` for the `v` prefix itself. Delete the `branches:`
  line if you copy that file.
- **`basic-package.yml` comments a step "Upload to Codecov" while setting only
  `run-coverage: true`.** `upload-coverage` defaults to `false`, so coverage is
  collected and never uploaded. Set `upload-coverage: true` if you meant to
  upload.

The example files are left as they are on purpose — `examples/` is shipped
product and is not edited by a documentation change.

## Customising

```yaml
with:
  run-tests: false # lint only
  pytest-args: '-m "not slow"' # narrow the test run
  upload-coverage: true # actually send coverage to Codecov
  use-nox-build: false # uv build --wheel instead of nox
```

Full input tables: [workflows.md](workflows.md).

To add steps of your own, depend on the reusable job:

```yaml
jobs:
  ci:
    uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@v1
    secrets: inherit

  extra-checks:
    needs: ci
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: EWS-Consulting-Public/ews-ci-action/.github/actions/setup-ews-ci@v1
        with:
          ews-credentials: ${{ secrets.EWS_CREDENTIALS }}
      - run: uv run my-check
```

For a step inside your own job, call the composite action directly —
[action-reference.md](action-reference.md).

## A complete working example

[ews-jupyter](https://github.com/EWS-Consulting-Private/ews-jupyter) is the
reference consumer; its CI and release workflows are the shape above in
production. The repository is private.
