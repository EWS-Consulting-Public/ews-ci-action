---
status: as-built
covers: the data-plan input - what is cached, what the key is, and why plan-ensure always runs
last-verified: 2026-09-05
---

# ADR 0006 — `data-plan:` caches the whole data root, keyed on the plan's lock

**Date:** 2026-09-05
**Status:** accepted

## Context

EWS packages keep large test data out of git: a *dataset document* in the
repository describes objects held in a cloud bucket, a *plan* selects the
subset a job needs, and `ews-gcp plan-ensure <plan>` fetches exactly that
selection to a local data root. A CI job that runs `plan-ensure` cold pays the
fetch on every run, and the cost follows the **number** of objects far more
than their size — sequential object round trips, not bandwidth. A job that
runs it against a directory already holding the bytes pays under a second.

Two facts decide the shape of a cache entry:

- **The key must move exactly when the bytes move.** A plan's lock
  (`<plan>.lock.json`, written by `ews-gcp plan-lock` and committed) carries
  one row per selected object with only what identifies its bytes — no prose,
  no author comments, no key order. The document itself carries all of those,
  so keying on the document evicts the cache when a sentence is edited.
- **The path must be known before anything is installed.** `actions/cache`
  needs a literal directory. The composite action already exports
  `EWS_DATA_ROOT=$GITHUB_WORKSPACE/.ews-data` to the job, and a dataset
  document bound to that variable lands its files under it.

## Decision

One optional string input on `ci.yml`, `data-plan`, the path of a plan in the
checkout. When set, the `test` job:

1. derives the lock path from the plan path and fails if either is missing,
   with the command that writes the lock;
2. restores `actions/cache` on `$GITHUB_WORKSPACE/.ews-data` under
   `ews-data-<OS>-<sha256(lock)[:16]>`, with `ews-data-<OS>-` as the prefix
   fallback;
3. runs `uv run ews-gcp plan-ensure <plan>` **unconditionally** and logs the
   seconds it took beside whether the restore was a hit or a miss;
4. saves the cache under the exact key when the restore was not an exact hit —
   **before** pytest.

The cache is the **whole data root**, not the plan's own paths. A plan's paths
are host-dependent and only computable once `ews-gcp` is installed and the
root is bound; the root is one literal path the action controls. Caching more
than a plan needs costs bytes on the cache, never correctness, because
`plan-ensure` decides what is correct.

## Consequences

- **`plan-ensure` is the safety gate, the cache is only a shortcut.** The
  prefix fallback deliberately hands back a stale tree when the data moved;
  `plan-ensure` turns it into the right one by fetching only what differs. A
  cache is never allowed to be the thing that decides the fixtures are
  correct, which is why the fetch step has no `if: cache-hit != 'true'`.
- **Saving before the tests** means one failing test does not cost the next
  run a cold fetch. `cache-hit` is `'true'` only on an exact key match, so a
  prefix restore still saves a fresh entry under the new key.
- **The consuming repository owns three things:** `ews-gcp-utils` in its
  dependencies, a committed lock kept current by its own hook, and a document
  bound to `EWS_DATA_ROOT`. A document bound to another variable is not
  cached by this input; such a repository calls the composite action directly
  and writes its own cache steps.
- **One plan per caller.** Two jobs reading different selections of the same
  data are two callers, or a hand-written job. `ci.yml` does not take a list.
- **The seconds are in the log and the job summary**, so the hit/miss and
  the fetch time of every run are readable without opening the cache tab.
- **The GitHub cache has a size budget per repository** and evicts oldest
  entries; a plan larger than that budget is never a hit. That is a fact
  about the plan, and the answer is a smaller plan.
- **The lint and build jobs never fetch data.** Only tests read it.

## Evidence

`ci.yml`, the four steps between the composite action and `Run tests` in the
`test` job, each guarded by `if: inputs.data-plan != ''`.

The lock-as-key argument and the timings that motivate caching at all were
measured in `ews-gcp-utils`'s own documentation before this input existed;
this ADR takes them as given.

## Not verified

The first observed run is recorded in [../workflows.md](../workflows.md)
§ *Not verified*. Cache eviction behaviour under the repository budget was not
exercised.

## Related

- [ADR 0003](0003-dataset-cache-redirected-into-the-workspace.md) — the other
  data location the action exports, for the release-asset loader
- [../workflows.md](../workflows.md) § *The data plan* — the consumer-facing
  description
