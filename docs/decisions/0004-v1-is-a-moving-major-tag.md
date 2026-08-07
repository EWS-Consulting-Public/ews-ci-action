---
status: as-built
covers: the floating v1 tag convention and what it means for consumers
last-verified: 2026-08-07
---

# ADR 0004 — `v1` is a moving major-version tag

**Date:** convention inherited from GitHub Actions practice; recorded
2026-08-07
**Status:** accepted

## Context

A `uses:` reference resolves a git ref at run time — a branch, a tag, or a
SHA. There is no package manager, no lockfile and no version range. Whatever a
consumer writes is exactly what runs.

That leaves three options, and every published GitHub Action picks one:

- **Pin a SHA.** Immutable and auditable; every consumer must open a PR for
  every fix.
- **Track a branch** (`@main`). Every push reaches every consumer immediately,
  including the broken ones.
- **Track a major tag** (`@v1`) that maintainers force-move forward.

## Decision

**Consumers reference `@v1`, and `v1` is force-updated to point at the newest
`v1.x` release.** Exact tags (`v1.2.0`) remain for anyone who wants to pin.

```bash
git tag -a v1.2.0 -m "…"
git push origin v1.2.0
git tag -fa v1 -m "Update v1 to v1.2.0"
git push --force origin v1
```

A breaking change gets `v2` and a new `v2` pointer; `v1` stays where it is so
existing consumers keep working.

Every reference in this repository's own examples and workflows uses `@v1`,
including the workflows' internal calls to their own composite action
(`ci.yml:107`, `:132`, `:162`, `release.yml:44`).

## Consequences

- **A fix reaches every consuming repository with no PR anywhere.** This is the
  property being bought, and it is the reason the repository exists — it was
  built to remove duplicated workflow YAML, which a per-consumer pin would
  reintroduce as per-consumer version bumps.
- **So does a regression.** There is no staged rollout and no way to hold a
  consumer back except by editing that consumer to pin an exact tag. Hence the
  rule in [../releasing.md](../releasing.md): test against more than one
  consuming repository, on a branch ref, before moving `v1`.
- **Force-pushing a tag is normal here.** That is unusual and worth stating
  explicitly, because it is otherwise a thing not to do.
- **`ci.yml` and `release.yml` call `setup-ews-ci@v1`, not a relative path.**
  A workflow on a branch therefore still loads the composite action *from the
  `v1` tag*, not from the branch being tested. Testing a change to `action.yml`
  by pointing a consumer at `…/ci.yml@my-branch` will silently exercise the old
  action. Point the consumer at the composite action directly, or accept that
  the two must be tested separately.
- **The tags carry no changelog.** There is no `CHANGELOG.md` and the tag
  messages are one line. What changed between two `v1.x` tags is only in the
  git log.

## Current state

Tags on this repository, 2026-08-07:

```console
$ git tag -l
v1
v1.0
```

Both point at `952e37c`, and both are **lightweight**, not annotated — the
`-a` / `-fa` form documented above was not used to create them. Nothing depends
on the distinction today; `uses:` resolves either.

## Related

- [../releasing.md](../releasing.md) — the tagging procedure and what counts as
  breaking
- [../adopting.md](../adopting.md) — how consumers write the reference
