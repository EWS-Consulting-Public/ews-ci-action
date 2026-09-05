# ews-ci-action

Reusable GitHub Actions CI/CD for Python packages built with
[uv](https://github.com/astral-sh/uv) and published to a private GitLab
package registry.

One composite action and two reusable workflows. A consuming repository
replaces its lint/test/build/publish YAML with a handful of lines.

```yaml
# .github/workflows/ci.yml
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

That gives you a skip check, a `ruff` lint job, a `pytest` matrix with
coverage, and a wheel build that runs only after both pass.

## What is in here

| Path | What it is |
| --- | --- |
| [`.github/actions/setup-ews-ci`](.github/actions/setup-ews-ci/action.yml) | Composite action: installs uv + Python, exports every `EWS_CREDENTIALS` key and the data root to the job, writes registry auth onto the runner, optionally `uv sync` |
| [`.github/workflows/ci.yml`](.github/workflows/ci.yml) | Reusable workflow: `check-skip` → `lint` → `test` (matrix, with an optional cached dataset plan) → `build` |
| [`.github/workflows/release.yml`](.github/workflows/release.yml) | Reusable workflow: on a green tagged CI run, build, publish to the registry, create a GitHub release |
| [`examples/`](examples/) | Copy-paste caller workflows |

Both workflows are `workflow_call`-only. They do not run on this repository's
own pushes.

## Requirements for a consuming repository

- A `pyproject.toml` and a uv-resolvable project. A committed `uv.lock` is
  used with `--frozen` when present.
- `ruff`, `pytest` and (for `use-nox-build`, the default) a `nox` session
  named `build` available through `uv`.
- An `EWS_CREDENTIALS` repository secret — a single JSON object, every key of
  which the action exports to the job as an environment variable. See
  [docs/credentials.md](docs/credentials.md).
- Optionally, a data plan: `data-plan: config/plans/<name>.yaml` fetches the
  tests' dataset once and caches it on the plan's committed lock. See
  [docs/workflows.md](docs/workflows.md) § *The data plan*.
- Linux runners. Every step is `shell: bash` and writes to `~/.config`.

## Versioning

Reference the workflows and the action by major version:

```yaml
uses: EWS-Consulting-Public/ews-ci-action/.github/workflows/ci.yml@v1
```

`v1` is a **moving tag** that is force-updated to the newest `v1.x`, so
consumers get fixes without editing anything. Pin an exact tag instead if you
need immutability. See
[ADR 0004](docs/decisions/0004-v1-is-a-moving-major-tag.md).

## Documentation

Start at [docs/README.md](docs/README.md).

| Page | Answers |
| --- | --- |
| [docs/adopting.md](docs/adopting.md) | How do I put this in my repository? |
| [docs/action-reference.md](docs/action-reference.md) | What does the composite action do to the runner? |
| [docs/workflows.md](docs/workflows.md) | Every input of `ci.yml` and `release.yml` |
| [docs/credentials.md](docs/credentials.md) | What goes in `EWS_CREDENTIALS`, and what is written where |
| [docs/releasing.md](docs/releasing.md) | Changing and tagging this repository |
| [docs/decisions/](docs/decisions/) | Why it is built this way |

## License

MIT — see [LICENSE](LICENSE).
