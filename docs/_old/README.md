---
status: reference
covers: the parked pre-2026-08-07 documentation
last-verified: 2026-08-07
---

# `docs/_old/` — parked documentation

**This is a parking area, not the documentation.** Read
[`../README.md`](../README.md) instead.

Everything here is the documentation as it stood **before 2026-08-07**, when
this repository was re-instrumented to the shape the rest of the estate uses
(`AGENTS.md` + a thin `CLAUDE.md` + a `docs/` map with YAML status headers and
`docs/decisions/` ADRs). **Nothing was deleted.** The new docs reformulate,
shorten and correct; this directory is what makes it checkable that no fact was
lost on the way.

| Parked file | Was | Replaced by |
| --- | --- | --- |
| [root-README.md](root-README.md) | `README.md` (repo root) | `README.md`, [`../action-reference.md`](../action-reference.md), [`../workflows.md`](../workflows.md), [`../credentials.md`](../credentials.md), [`../releasing.md`](../releasing.md) |
| [QUICKSTART.md](QUICKSTART.md) | `QUICKSTART.md` (repo root) | [`../adopting.md`](../adopting.md), and the troubleshooting section of [`../credentials.md`](../credentials.md) |
| [CONTRIBUTING.md](CONTRIBUTING.md) | `CONTRIBUTING.md` (repo root) | [`../releasing.md`](../releasing.md), [ADR 0004](../decisions/0004-v1-is-a-moving-major-tag.md) |
| [examples-README.md](examples-README.md) | `examples/README.md` | [`../adopting.md`](../adopting.md) § *Examples* and § *Customising* |

`examples/README.md` was not replaced in place. The three example YAML files
remain in [`../../examples/`](../../examples/); what they are and when to use
each is now in `adopting.md`, so the guidance sits next to the workflow input
tables rather than in a separate index that drifted from the directory.

**The relative links inside these files no longer resolve.** They were written
for the repository root and `examples/`, and moving them down two levels broke
the paths — on top of the four that pointed at files which never existed
(below). They are left exactly as they were rather than repaired: the point of
a parked file is that it is verbatim.

## Do not cite these files

Claims found **wrong against the YAML on 2026-08-07** while rewriting. Each was
checked against `.github/`.

| Parked claim | Where | What the YAML does |
| --- | --- | --- |
| The composite action "Configures GitLab Package Registry (via `setup-uv-gitlab-registry-action`)"; release.yml "Uses `setup-uv-gitlab-registry-action` with upload token" | `root-README.md` § *Composite Action*, § *release.yml* | That action is not used anywhere. The registry is configured inline by writing `~/.netrc` and `~/.config/uv/uv.toml` in the setup script. `grep -rn 'setup-uv-gitlab-registry' .github/` → no output |
| Release step 4: "Downloads build artifacts from CI" | `root-README.md` § *release.yml* | It rebuilds — `uvx nox -s build` (`release.yml:60-61`). No `download-artifact` step exists in the repository. Recorded as [ADR 0005](../decisions/0005-release-rebuilds-rather-than-downloading.md) |
| `ews-credentials-keys` performs "validation" of the credentials JSON | `root-README.md`, `QUICKSTART.md` throughout | The input is declared, forwarded through four layers and **read by nothing**. `grep -rn 'EWS_CREDENTIALS_KEYS' .github/` returns only the declaration and the forwarding. The QUICKSTART troubleshooting entry for `"Missing required keys in EWS_CREDENTIALS"` describes an error message no code emits |
| Examples: `basic-ci.yml`, `release.yml`, `custom-steps.yml`, `ews-jupyter-migration.md` | `root-README.md` § *Examples*, § *Next Steps* | None of these files exists. `examples/` contains `basic-package.yml`, `matrix-testing.yml`, `release-basic.yml`. Four broken links |
| Examples: `custom-steps.yml`, `release-only.yml` | `examples-README.md` § *Available Examples* | Same — neither exists, and `release-basic.yml` (which does) is not listed |
| Input `gitlab-project-id: "12345678"` | `examples-README.md` § *Different GitLab Project* | No such input on any workflow or action. The registry URL comes from the `EWS_CREDENTIALS` JSON |
| Input `python-version` on the CI workflow | `QUICKSTART.md` § *Next Steps*, `examples-README.md` § *Change Python Version* | `ci.yml` takes `python-versions` (a JSON array) and `lint-python-version`. `python-version` is an input of the *composite action* and of `release.yml`, not of `ci.yml` |
| CI workflow inputs list omits `upload-coverage`, `pytest-args`, `ews-credentials-keys`; says `run-coverage` "Collect coverage and upload to Codecov" | `root-README.md` § *ci.yml* | Three inputs were undocumented. And `run-coverage` only collects — uploading needs `upload-coverage`, which defaults to `false`, so the documented behaviour was the opposite of the default |
| Testing a change: point a consumer at `EWS-Consulting-Private/ews-ci-action/...@branch` | `CONTRIBUTING.md:24` | Wrong owner. This repository is `EWS-Consulting-Public/ews-ci-action`; a `Private` path does not resolve |
| Matrix testing supports "different OS platforms (Linux, macOS, Windows)" | `examples-README.md` § *matrix-testing.yml* | Every job is `runs-on: ubuntu-latest` and the matrix has one dimension, `python-version`. There is no OS input. The composite action is `shell: bash` with `~/.config` paths throughout |
| "CalVer-aware release automation" | `root-README.md` § *Features* | Nothing in either workflow knows about CalVer. The release trigger tests only that the tag starts with `v`. Version bumping happens in the consuming repository |
| `make release  # Or: nox -s release` | `QUICKSTART.md` § *Step 5* | Describes the consuming repository's own tooling, not anything this repository provides. Restated in [`../adopting.md`](../adopting.md) as "tag a version and push the tag", which is the part that matters here |

Two further notes rather than errors:

- `root-README.md` § *Migration Guide* has a broken code fence. A stray
  four-backtick fence opens at line 235 and closes at 258, so everything
  between — including the `**After**:` heading and the nested ` ```yaml ` fence
  at 239 — renders as literal code on GitHub.
- `root-README.md`'s "218 lines → 46 lines (79% reduction)" figure for the
  reference consumer was not re-measured. The consumer repository is private
  and was not opened for this pass.

## When this directory can go

It is kept so a review pass can confirm nothing worth keeping was dropped.
**Once that review has happened this whole directory can be deleted** — git
holds the history.
