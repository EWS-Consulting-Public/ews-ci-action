---
status: as-built
covers: the setup-ews-ci composite action - inputs and every file it writes
last-verified: 2026-08-07
---

# `setup-ews-ci` — what it does to the runner

Source: [`.github/actions/setup-ews-ci/action.yml`](../.github/actions/setup-ews-ci/action.yml).

Use it directly when the reusable workflows do not fit:

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: EWS-Consulting-Public/ews-ci-action/.github/actions/setup-ews-ci@v1
    with:
      python-version: "3.12"
      ews-credentials: ${{ secrets.EWS_CREDENTIALS }}

  - run: uv run pytest
```

## Inputs

| Input | Default | Effect |
| --- | --- | --- |
| `python-version` | `"3.12"` | Passed to `uv python install` |
| `install-dependencies` | `"true"` | Run the `uv sync` step. String, not boolean — quote it |
| `ews-credentials` | `''` | The `EWS_CREDENTIALS` JSON object. Without it, every credential-derived file below is skipped |
| `ews-credentials-keys` | `''` | **Declared and never read.** See below |

`ews-credentials-keys` is exported into the setup step's environment as
`EWS_CREDENTIALS_KEYS` (`action.yml:38`) and no script reads it. The
"validation" its description promises is not implemented. Setting it is
harmless; omitting it loses nothing.

## Steps, in order

1. **`astral-sh/setup-uv@v7`** with `enable-cache: true`.
2. **`uv python install <python-version>`**.
3. **The setup script** — a single `bash` block, detailed below.
4. **`uv sync`**, only if `install-dependencies == 'true'`: `uv sync --frozen
   --all-extras` when `uv.lock` exists, otherwise `uv sync --all-extras`.

Note `--all-extras`: optional dependency groups are installed, so a package
whose tests need an extra does not have to ask for it.

## What the setup script writes

Every file below is written **only if** its inputs are non-empty. A missing
credential prints an informational line and continues; nothing fails.

| File | Written when | Contents | Mode |
| --- | --- | --- | --- |
| `~/.config/ews/config/github.toml` | `github.token` is present (always, in practice) | `[github]` with the actor, a no-reply address, and the job's `GITHUB_TOKEN` | `600` |
| `~/.netrc` | `gitlab_api_read_token` **and** `gitlab_package_registry_url` | `machine gitlab.com`, login `__token__` — HTTP basic auth for uv/pip | `600` |
| `~/.config/uv/uv.toml` | same condition | An `[[index]]` named `gitlab` pointing at `<registry-url>/simple`, `authenticate = "always"`, `ignore-error-codes = [403]` | default |
| `~/.pypirc` | `gitlab_api_token` **and** `gitlab_package_registry_url` | A `[gitlab]` distutils target for twine | `600` |
| `~/.config/ews/config/ammonit-or.toml` | `ammonit_or_password` | `[ammonit-or]` section | `600` |
| `~/.config/ews/config/windcube-insights.toml` | `windcube_insights_password` | `[windcube-insights]` section | `600` |

It also creates `~/.config/ews/dataset_registries`, `~/.config/ews/config`,
`~/.config/uv`, `~/.cache/ews/datasets` and
`$GITHUB_WORKSPACE/.cache/ews/datasets`, and finally copies
`~/.config/ews/config/*.toml` up into `~/.config/ews/` — a compatibility shim
the source labels "for old tests" (`action.yml:167-170`).

`EWS_DATASETS_SYSTEM_CACHE_PATH` is set to `$GITHUB_WORKSPACE/.cache/ews/datasets`
for the duration of the step
([ADR 0003](decisions/0003-dataset-cache-redirected-into-the-workspace.md)).

## The two GitLab tokens are not interchangeable

- `gitlab_api_read_token` → `~/.netrc` + `~/.config/uv/uv.toml`. This is the
  **install** path: it lets `uv sync` resolve private wheels.
- `gitlab_api_token` → `~/.pypirc`. This is the **publish** path, used by
  `twine` / `nox -s publish` during a release.

A repository that only needs to install dependencies needs the read token
alone. Supplying only the write token configures publishing and leaves
dependency resolution unauthenticated.

## Failure modes

The script is written to degrade rather than fail, which means a
misconfiguration surfaces later and less clearly:

- **No `EWS_CREDENTIALS`** — prints `⚠️ No EWS_CREDENTIALS provided`, writes no
  registry config, and the job fails later in `uv sync` with a resolution or
  `401` error against the private index.
- **Registry URL present, read token missing** (or vice versa) — prints
  `ℹ️ … skipping package registry setup` and behaves as above. Both keys are
  required together.
- **`EWS_CREDENTIALS` is not valid JSON** — `jq -r '… // empty'` yields empty
  strings for every key, so this looks exactly like "no credentials provided".
  There is no parse check.

## Platform

Every step is `shell: bash` and the paths are POSIX (`~/.config`, `~/.netrc`).
It is written for `ubuntu-latest`. It has not been exercised on Windows or
macOS runners, and `~/.config/uv/uv.toml` is not where uv looks on Windows.
