---
status: as-built
covers: why release.yml builds its own wheel instead of using CI's artifact
last-verified: 2026-08-07
---

# ADR 0005 — The release workflow rebuilds the wheel

**Date:** recorded 2026-08-07 from `release.yml`
**Status:** accepted — recorded because the parked docs claimed the opposite

## Context

`ci.yml`'s `build` job produces a wheel and uploads it as a `dist` artifact
with a 7-day retention (`ci.yml:177-183`). `release.yml` is triggered by that
same workflow run completing successfully, so the artifact is available to it
via `actions/download-artifact` and the `workflow_run` event's run id.

Using it would be the conventional build-once-promote-many shape: the bytes
that were tested are the bytes that ship.

## Decision

**`release.yml` does not download the artifact. It checks out the tag and
builds again** (`release.yml:60-61`):

```yaml
- name: Build package
  run: uvx nox -s build
```

There is no `actions/download-artifact` step anywhere in this repository —
verified 2026-08-07 by `grep -rn 'download-artifact' .github/`, which returns
nothing.

## Consequences

- **The published wheel is not the tested wheel.** It is a wheel built from the
  same tag, which is not the same guarantee. For a pure-Python wheel built from
  a committed tree with a committed `uv.lock` this is very likely
  byte-equivalent, but nothing checks it, and a build that is not reproducible
  (an embedded timestamp, a resolved-at-build-time dependency) would differ
  silently.
- **The release path does not depend on artifact retention or on `actions: read`
  reaching the artifact API.** It only needs the tag. That is genuinely simpler
  and removes a class of failure where a release cannot find its input.
- **`nox -s build` is required even by repositories that set
  `use-nox-build: false` in CI.** The CI input toggles between `nox -s build`
  and `uv build --wheel` (`ci.yml:169-175`); the release step has no such
  toggle. A repository that opted out of nox for CI will still fail at release
  time without the session.
- **A release can succeed for a tag whose CI artifact was never uploaded** —
  for example with `upload-artifacts: false`. The two paths are independent.
- **`uv.lock` is generated if absent** (`release.yml:51-58`) and then attached
  to the GitHub release. For a repository without a committed lockfile, the
  lock published alongside the wheel is resolved at release time and is not the
  one CI resolved.

## Evidence

`release.yml:39-61` — checkout at `github.event.workflow_run.head_branch`,
setup with `install-dependencies: "false"`, conditional `uv lock`, then
`uvx nox -s build`.

`grep -rn 'download-artifact' .github/` → no output (2026-08-07).

The parked [`docs/_old/root-README.md`](../_old/root-README.md) § *release.yml*
listed "Downloads build artifacts from CI" as step 4 of what the workflow does.
That was wrong when written or has been wrong since; this ADR exists so the
claim is not reintroduced.

## Related

- [../workflows.md](../workflows.md) — the full release step list
- [../_old/README.md](../_old/README.md) — the other corrected claims
