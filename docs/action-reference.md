---
status: as-built
covers: the setup-ews-ci composite action - inputs, what it exports to the job, and every file it writes
last-verified: 2026-09-05
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
      ews-credentials-keys: ${{ vars.EWS_CREDENTIALS_KEYS }}

  - run: uv run pytest
```

## Inputs

| Input | Default | Effect |
| --- | --- | --- |
| `python-version` | `"3.12"` | Passed to `uv python install` |
| `install-dependencies` | `"true"` | Run the `uv sync` step. String, not boolean — quote it |
| `ews-credentials` | `''` | The `EWS_CREDENTIALS` JSON object. Without it, nothing credential-derived is exported or written |
| `ews-credentials-keys` | `''` | The `EWS_CREDENTIALS_KEYS` JSON array. When given, the step **fails** if the object lacks any key it names |

## Steps, in order

1. **`astral-sh/setup-uv@v7`** with `enable-cache: true`.
2. **`uv python install <python-version>`**.
3. **The setup script** — a single `bash` block, detailed below.
4. **`uv sync`**, only if `install-dependencies == 'true'`: `uv sync --frozen
   --all-extras` when `uv.lock` exists, otherwise `uv sync --all-extras`.

Note `--all-extras`: optional dependency groups are installed, so a package
whose tests need an extra does not have to ask for it.

## What the setup script does

The block runs under `set -euo pipefail`; the parts that can fail, fail the
job with a `::error::` line naming what to fix.

### 1. The GitHub token

`~/.config/ews/config/github.toml` is written from the job's own
`github.token`, with the actor as username. This is not part of
`EWS_CREDENTIALS`.

### 2. Every credential key becomes a job-wide environment variable

`EWS_CREDENTIALS` is a JSON object. The script:

1. **Rejects it** if it is not a JSON object.
2. **Validates it** against `EWS_CREDENTIALS_KEYS` when that input is
   non-empty: every key the array names must be present in the object, or the
   step fails listing the missing ones. The two are written together by the
   EWS credential tooling ([credentials.md](credentials.md) § *Setting it*),
   so a mismatch means one of them is stale.
3. **Exports every key** whose value is non-empty to `$GITHUB_ENV` as
   `UPPER(key)` with `-` turned into `_`, so it is visible to **every later
   step of the job**, not only this one. Each value is passed through
   `::add-mask::` first, line by line, because a parsed value is a fresh string
   that GitHub's masking of the whole secret does not cover.

The naming is not a convention invented here. The payload's keys are
`<section>_<field>` of the EWS configuration layer, and `UPPER(key)` is exactly
the *flat environment name* that layer reads when no TOML file sets the field —
`gcp_default_key` → `GCP_DEFAULT_KEY` fills `[gcp].default_key`,
`gitlab_api_read_token` → `GITLAB_API_READ_TOKEN` fills
`[gitlab].api_read_token`. So any `ews-*` tool run in a later step resolves
every credential section from the environment with no config file written,
and **adding a key to the secret makes it reachable on the runner without
touching this action**.

An empty value is not exported (an informational line says so). A key that is
not a valid environment variable name is skipped with a warning.

### 3. The data root and the dataset cache

Two more variables are exported to the whole job, both pointing inside the
workspace — the one path a runner guarantees writable:

| Variable | Value | Read by |
| --- | --- | --- |
| `EWS_DATA_ROOT` | `$GITHUB_WORKSPACE/.ews-data` | The EWS dataset documents and plans — it fills `[ews-data].root`. This is the directory `ci.yml`'s `data-plan:` input caches |
| `EWS_DATASETS_SYSTEM_CACHE_PATH` | `$GITHUB_WORKSPACE/.cache/ews/datasets` | The release-asset dataset loader ([ADR 0003](decisions/0003-dataset-cache-redirected-into-the-workspace.md)) |

Both directories are created. Both are inside the checkout, so a working-tree
cleanliness check in a consuming repository sees them unless they are ignored.

### 4–6. The files

Every file below is written **only if** its inputs are non-empty. A missing
credential prints an informational line and continues.

| File | Written when | Contents | Mode |
| --- | --- | --- | --- |
| `~/.config/ews/config/github.toml` | `github.token` is present (always, in practice) | `[github]` with the actor, a no-reply address, and the job's `GITHUB_TOKEN` | `600` |
| `~/.netrc` | `gitlab_api_read_token` **and** `gitlab_package_registry_url` | `machine gitlab.com`, login `__token__` — HTTP basic auth for uv/pip | `600` |
| `~/.config/uv/uv.toml` | same condition | An `[[index]]` named `gitlab` pointing at `<registry-url>/simple`, `authenticate = "always"`, `ignore-error-codes = [403]` | default |
| `~/.pypirc` | `gitlab_api_token` **and** `gitlab_package_registry_url` | A `[gitlab]` distutils target for twine | `600` |
| `~/.config/ews/config/ammonit-or.toml` | `ammonit_or_password` | `[ammonit-or]` section | `600` |
| `~/.config/ews/config/windcube-insights.toml` | `windcube_insights_password` | `[windcube-insights]` section | `600` |

It also creates `~/.config/ews/dataset_registries`, `~/.config/ews/config`,
`~/.config/uv` and `~/.cache/ews/datasets`, and finally copies
`~/.config/ews/config/*.toml` up into `~/.config/ews/` — a compatibility shim
the source labels "for old tests".

The GitLab files are a convenience for the two tools that do not read the EWS
configuration layer (`uv`'s index auth and `twine`); every EWS tool reads the
exported variables directly.

## The two GitLab tokens are not interchangeable

- `gitlab_api_read_token` → `~/.netrc` + `~/.config/uv/uv.toml`. This is the
  **install** path: it lets `uv sync` resolve private wheels.
- `gitlab_api_token` → `~/.pypirc`. This is the **publish** path, used by
  `twine` / `nox -s publish` during a release.

A repository that only needs to install dependencies needs the read token
alone. Supplying only the write token configures publishing and leaves
dependency resolution unauthenticated.

## Failure modes

Three things fail the step outright, each with an `::error::` line:

- **`EWS_CREDENTIALS` is not a JSON object.** Before 2026-09-05 this looked
  exactly like "no credentials provided"; now it is an error, because a secret
  pasted with a stray character is a misconfiguration, not an absence.
- **`EWS_CREDENTIALS_KEYS` names a key the object does not carry.** The error
  lists the missing keys and says to re-run the credential tooling.
- **`EWS_CREDENTIALS_KEYS` is not a JSON array.**

The rest still degrades rather than fails, so a misconfiguration surfaces later
and less clearly:

- **No `EWS_CREDENTIALS`** — prints `⚠️ No EWS_CREDENTIALS provided`, exports
  nothing, writes no registry config, and the job fails later in `uv sync`
  with a resolution or `401` error against the private index.
- **Registry URL present, read token missing** (or vice versa) — prints
  `ℹ️ … skipping package registry setup` and behaves as above. Both keys are
  required together.
- **A key present with an empty value** — not exported; the line
  `ℹ️ <key> is empty, not exported` says so.

## Platform

Every step is `shell: bash` and the paths are POSIX (`~/.config`, `~/.netrc`).
It is written for `ubuntu-latest`, whose runner image ships `jq`. It has not
been exercised on Windows or macOS runners, and `~/.config/uv/uv.toml` is not
where uv looks on Windows.

## Not verified

The script's credential block was exercised locally on 2026-09-05 (five
payload cases: valid, missing key, invalid JSON, no secret, no keys array) by
extracting it from the YAML and running it under a sandbox `HOME` and
`GITHUB_ENV`. The `::add-mask::` behaviour and `$GITHUB_ENV` propagation are
GitHub's; they were not observed here.
