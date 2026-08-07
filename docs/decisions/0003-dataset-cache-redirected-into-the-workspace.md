---
status: as-built
covers: the EWS_DATASETS_SYSTEM_CACHE_PATH override and its scope
last-verified: 2026-08-07
---

# ADR 0003 — The dataset system cache is redirected into the workspace

**Date:** recorded 2026-08-07 from `action.yml`
**Status:** accepted, **but see § Consequences — the override does not reach
later steps**

## Context

EWS packages that read datasets resolve a *system* cache directory — a
machine-wide path, outside any one user's home, chosen so that several users on
a shared host share one download. On a developer workstation or an EWS server
that location exists and is writable by the right group.

A GitHub Actions runner has neither property. The job runs as an unprivileged
user, and a machine-wide path is either absent or not writable, so a package
that tries to populate its system cache fails on a permission error rather than
on anything to do with the code under test.

## Decision

Override the location with an environment variable pointing inside the
workspace, and create the directory in advance:

```yaml
env:
  # Override system cache to user-writable location in CI
  EWS_DATASETS_SYSTEM_CACHE_PATH: ${{ github.workspace }}/.cache/ews/datasets
```

— `action.yml:40-41`. The setup script then creates both
`~/.cache/ews/datasets` and `$GITHUB_WORKSPACE/.cache/ews/datasets`
(`action.yml:48-49`).

`${{ github.workspace }}` is chosen over `$HOME` because it is the one path a
runner guarantees is writable and is where the checkout already lives.

## Consequences

- **The override is scoped to the setup step only.** It is declared under that
  step's `env:` key and is **never written to `$GITHUB_ENV`** — verified
  2026-08-07: `grep -n 'GITHUB_ENV' action.yml` returns nothing. So a later
  step in the same job, such as `uv run pytest`, does **not** see
  `EWS_DATASETS_SYSTEM_CACHE_PATH`, and any package reading it at test time
  falls back to its own default.

  What survives the step is the *directory*, not the variable. If tests pass
  today, it is because the package tolerates the default path, or because
  `~/.cache/ews/datasets` — which is also created, and is user-writable — is
  what it actually uses. **The variable as written protects nothing beyond the
  setup step.** Making it effective would mean one line:
  `echo "EWS_DATASETS_SYSTEM_CACHE_PATH=…" >> $GITHUB_ENV`.

- **Nothing in this repository consumes the variable**, so its name and
  semantics are owned by the EWS dataset package, not here. Renaming it there
  silently disables this override.

- **The workspace cache is not persisted between jobs.** It is not part of the
  uv cache that `astral-sh/setup-uv@v7` restores, and it is not uploaded. Each
  job re-downloads whatever it needs.

- **It is inside the checkout.** `$GITHUB_WORKSPACE/.cache/` will appear as an
  untracked directory to any step that inspects the working tree — a
  `git status --porcelain` cleanliness check in a consuming repository would
  see it unless `.cache/` is ignored.

## Evidence

`action.yml:40-41` — the `env:` declaration and the comment stating the intent.

`action.yml:45-49` — the five `mkdir -p` calls, including both cache
directories.

No `$GITHUB_ENV` or `$GITHUB_PATH` write anywhere in the file
(`grep -n 'GITHUB_ENV\|GITHUB_PATH' action.yml` → no output, 2026-08-07).

## Not verified

Which EWS package reads `EWS_DATASETS_SYSTEM_CACHE_PATH`, what its default is,
and whether any consuming repository's tests actually exercise the dataset
cache. None of that is in this repository, and no run was observed.

## Related

- [../action-reference.md](../action-reference.md) — the full list of what the
  step creates
- [ADR 0001](0001-composite-action-not-javascript.md) — why this is a bash
  block rather than typed code
