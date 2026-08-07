---
status: as-built
covers: why setup-ews-ci is a composite action and what that costs consumers
last-verified: 2026-08-07
---

# ADR 0001 — The setup step is a composite action

**Date:** recorded 2026-08-07 from `action.yml`
**Status:** accepted

## Context

GitHub Actions offers three ways to package a reusable step:

- **JavaScript** — `using: node20`, a bundled `dist/index.js`, the toolkit
  libraries, and a build step that must be committed.
- **Docker** — `using: docker`, an image built or pulled per run.
- **Composite** — `using: composite`, an ordered list of the same steps a
  workflow author would write, executed in the caller's job.

This action's job is to install `uv`, install Python, and write six
configuration files into the runner's home directory
(`~/.netrc`, `~/.config/uv/uv.toml`, `~/.pypirc`, and three under
`~/.config/ews/config/`).

## Decision

**`using: composite`** (`action.yml:23`).

The whole implementation is one `astral-sh/setup-uv@v7` step, a
`uv python install`, one `shell: bash` block of roughly 130 lines, and a
conditional `uv sync`.

## Consequences

- **The side effects are files on the runner, not action outputs.** This action
  declares no `outputs:`. What it produces is state in `$HOME` that later steps
  in the same job pick up implicitly — `uv sync` reads `~/.config/uv/uv.toml`,
  `twine` reads `~/.pypirc`, EWS packages read `~/.config/ews/config/`. A
  consumer cannot query what was configured; they can only observe that a later
  step worked.
- **It runs per job, not per workflow.** `ci.yml` calls it in `lint`, `test`
  (once per matrix entry) and `build` — each a fresh runner needing its own
  copy of the setup. A four-entry matrix runs it six times.
- **It composes with third-party actions.** `astral-sh/setup-uv@v7` is a
  `uses:` step inside it, which a JavaScript action could not do without
  reimplementing uv installation and cache handling.
- **No build artifact to keep in sync.** A JavaScript action requires
  committing compiled output, and a stale `dist/` that does not match `src/` is
  the classic failure. Here the file that is read is the file that is edited.
- **The cost is that it is a shell script.** There is no type checking, no unit
  test, and no way to run it except on a runner. Errors are `set +e`-ish by
  construction: every credential block is an `if` that prints an informational
  message and continues, so a misconfiguration surfaces later, in `uv sync`,
  rather than at the point of failure.
- **It is Linux-only in practice.** `shell: bash` with `~/.config` paths. A
  JavaScript action would have been cross-platform for free. Nothing here has
  been exercised on a Windows or macOS runner.

## Evidence

`action.yml:22-24` — `runs: using: composite`. No `outputs:` key anywhere in
the file. Verified 2026-08-07.

`ci.yml:107`, `:132`, `:162` — three separate `uses:` of the composite action,
one per job.

## Related

- [../action-reference.md](../action-reference.md) — every file it writes
- [ADR 0002](0002-one-credentials-secret-not-many.md) — what it parses
- [ADR 0003](0003-dataset-cache-redirected-into-the-workspace.md) — the one
  environment variable it sets
