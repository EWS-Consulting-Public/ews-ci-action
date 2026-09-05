---
status: as-built
covers: the EWS_CREDENTIALS secret - its keys, the variable each one becomes on the runner, and which file some of them also land in
last-verified: 2026-09-05
---

# `EWS_CREDENTIALS`

One repository secret holds every credential the CI needs, as a single JSON
object. Individual keys are never separate secrets
([ADR 0002](decisions/0002-one-credentials-secret-not-many.md)).

```json
{
  "ammonit_or_password": "…",
  "gcp_default_key": "…",
  "gitlab_api_read_token": "…",
  "gitlab_api_token": "…",
  "gitlab_package_registry_id": "…",
  "gitlab_package_registry_url": "…",
  "windcube_insights_password": "…"
}
```

**Every key is exported to the job as `UPPER(key)`** — see
[action-reference.md](action-reference.md) § *Every credential key becomes a
job-wide environment variable*. That is the whole mechanism: the keys are
`<section>_<field>` of the EWS configuration layer, so the uppercased name is
the flat environment name that layer reads, and every `ews-*` tool in a later
step resolves the credential without a config file. The list above is what the
credential tooling writes today; the action does not know the keys by name and
a new one needs no change here.

Every key is optional to the action. An empty value is not exported and the
file it would feed is skipped.

## What each key is for

| Key | Becomes | Also feeds | Needed for |
| --- | --- | --- | --- |
| `gitlab_api_read_token` | `GITLAB_API_READ_TOKEN` | `~/.netrc`, `~/.config/uv/uv.toml` | **Installing** private wheels — `uv sync` |
| `gitlab_package_registry_url` | `GITLAB_PACKAGE_REGISTRY_URL` | both of the above, and `~/.pypirc` | Both paths; required alongside either token |
| `gitlab_api_token` | `GITLAB_API_TOKEN` | `~/.pypirc` | **Publishing** during a release |
| `gitlab_package_registry_id` | `GITLAB_PACKAGE_REGISTRY_ID` | — | The EWS GitLab tooling, when a test reaches the registry API |
| `gcp_default_key` | `GCP_DEFAULT_KEY` | — | **Fetching datasets** from the bucket — `ews-gcp plan-ensure`, and so `ci.yml`'s `data-plan:` input |
| `ammonit_or_password` | `AMMONIT_OR_PASSWORD` | `~/.config/ews/config/ammonit-or.toml` | Packages whose tests reach that data source |
| `windcube_insights_password` | `WINDCUBE_INSIGHTS_PASSWORD` | `~/.config/ews/config/windcube-insights.toml` | Same |

The read token and the write token are different tokens with different scopes.
See [action-reference.md](action-reference.md) § *The two GitLab tokens are not
interchangeable*.

The GitHub token is **not** in this secret — the composite action takes
`github.token` from the job context and writes it to
`~/.config/ews/config/github.toml` itself.

## Setting it

The secret and its companion variable are written by the EWS credential
tooling, never pasted into the GitHub UI:

```bash
uv run ews-github configure --org <owner> --repo <repo> \
  --skip-protection --skip-teams --skip-tag-protection --skip-merge-methods
```

See
[ews-github-utils](https://github.com/EWS-Consulting-Private/ews-github-utils)
for that command. It composes the payload from the host's own EWS
configuration — one row per declared credential — and writes two things on the
repository:

- the **`EWS_CREDENTIALS` secret** (Actions and Codespaces), the JSON object;
- the **`EWS_CREDENTIALS_KEYS` variable**, a JSON array of the object's keys.

**The action reads both.** When a caller forwards the variable
(`ews-credentials-keys: ${{ vars.EWS_CREDENTIALS_KEYS }}`), the setup step
fails if the secret lacks any key the array names. Since the same command
writes both, a failure there means one was updated without the other — re-run
the command.

## Rules for this repository

- **A credential value never appears in this repository.** Key names are the
  action's public interface and are necessarily here. Values, and any registry
  URL, host, bucket or account they would reveal, are not — this repository is
  public.
- Everything the action writes is `chmod 600` and lives on an ephemeral runner;
  everything it exports lives in that job's environment only.
- Values reach the runner through GitHub's secret masking, and the action masks
  each parsed value again before exporting it. Do not add a step that echoes
  one; masking is best-effort and a value split across lines or encodings can
  escape it.

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

or `secrets: inherit`.

**The setup step fails with `EWS_CREDENTIALS is not a JSON object`.** The
secret's value is not the payload the tooling writes — re-run the command in
§ *Setting it* rather than editing the secret by hand.

**The setup step fails naming missing keys.** The secret and the variable were
written at different times. Re-run the command; it rewrites both.

**A dataset fetch fails with a credential error.** Look for
`✅ Exported GCP_DEFAULT_KEY` in the setup step's log. If it is absent, the
secret predates that key — re-run the command.

**The release job never starts.** It requires a *successful* CI run whose
`head_branch` starts with `v`. Verify the tag matches, and that CI itself went
green — `check-skip` skipping every job still counts as success.
