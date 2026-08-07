---
status: as-built
covers: the EWS_CREDENTIALS secret - its keys, and which file each one lands in
last-verified: 2026-08-07
---

# `EWS_CREDENTIALS`

One repository secret holds every credential the CI needs, as a single JSON
object. Individual keys are never separate secrets
([ADR 0002](decisions/0002-one-credentials-secret-not-many.md)).

```json
{
  "gitlab_api_read_token": "…",
  "gitlab_api_token": "…",
  "gitlab_package_registry_url": "…",
  "ammonit_or_password": "…",
  "windcube_insights_password": "…"
}
```

Every key is optional. The setup script extracts each with
`jq -r '.<key> // empty'` and skips the file it feeds when the value is empty.

## What each key is for

| Key | Feeds | Needed for |
| --- | --- | --- |
| `gitlab_api_read_token` | `~/.netrc`, `~/.config/uv/uv.toml` | **Installing** private wheels — `uv sync` |
| `gitlab_package_registry_url` | both of the above, and `~/.pypirc` | Both paths; required alongside either token |
| `gitlab_api_token` | `~/.pypirc` | **Publishing** during a release |
| `ammonit_or_password` | `~/.config/ews/config/ammonit-or.toml` | Packages whose tests reach that data source |
| `windcube_insights_password` | `~/.config/ews/config/windcube-insights.toml` | Same |

The read token and the write token are different tokens with different scopes.
See [action-reference.md](action-reference.md) § *The two GitLab tokens are not
interchangeable*.

The GitHub token is **not** in this secret — the composite action takes
`github.token` from the job context and writes it to
`~/.config/ews/config/github.toml` itself.

## Setting it

The secret is normally pushed by the EWS credential tooling rather than pasted
into the GitHub UI:

```bash
uv run ews-github export-credentials --repo <your-repo> --set-secret --set-keys
```

See
[ews-github-utils](https://github.com/EWS-Consulting-Private/ews-github-utils)
for that command.

`--set-keys` additionally writes an `EWS_CREDENTIALS_KEYS` repository variable,
a JSON array of the keys that should be present. **Nothing in this repository
reads it** — see [README.md](README.md) § *Open questions*. It is accepted as an
input so that callers already passing it do not break.

## Rules for this repository

- **A credential value never appears in this repository.** Key names are the
  action's public interface and are necessarily here. Values, and any registry
  URL, host or account they would reveal, are not — this repository is public.
- Everything the action writes is `chmod 600` and lives on an ephemeral runner.
- Values reach the runner through GitHub's secret masking. Do not add a step
  that echoes a parsed value; `jq` output is not automatically masked.

## Troubleshooting

**`401 Unauthorized` or an unresolvable dependency during `uv sync`.**
The registry was not configured. Either the secret is missing, or
`gitlab_api_read_token` and `gitlab_package_registry_url` are not *both*
present — the action needs the pair. Check the setup step's log for
`ℹ️ No GitLab read token or package registry URL, skipping package registry setup`.

**The setup step logs `⚠️ No EWS_CREDENTIALS provided` even though the secret
is set.** A reusable workflow does not inherit secrets implicitly. The caller
must pass them:

```yaml
secrets:
  EWS_CREDENTIALS: ${{ secrets.EWS_CREDENTIALS }}
```

or `secrets: inherit`. The same line appears if the secret is set but its value
is not valid JSON, because every `jq` extraction then yields empty.

**The release job never starts.** It requires a *successful* CI run whose
`head_branch` starts with `v`. Verify the tag matches, and that CI itself went
green — `check-skip` skipping every job still counts as success.
