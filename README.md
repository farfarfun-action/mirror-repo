# mirror-repo

Mirror all (or a given list of) repos from one hosting platform's org to
another's — GitHub, Gitee, GitLab or GitCode, in any src/dst combination. A
thin wrapper around
[`farfarfun/funmirror`](https://github.com/farfarfun/funmirror), which
mirrors repos in parallel and skips any repo whose default branch already
has the same latest commit on both sides — no third-party mirror action
dependency.

If `repo-names` is not supplied, the action lists every repo in `src` itself
(requires `src_token`).

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `src` | yes | — | Source, format `<platform>/<org>`. platform is one of `github`/`gitee`/`gitlab`/`gitcode`, e.g. `github/kunpengcompute` |
| `dst` | yes | — | Destination, format `<platform>/<org>`. Same platform values as `src` |
| `src_token` | no | — | API token for the source platform, used to list repos (when `repo-names` is not provided) and to clone private repos |
| `dst_token` | no | — | API token for the destination platform, used to look up / create destination repos |
| `src_key` | no | — | SSH private key for the source platform, only required when the `src` platform is `gitee` |
| `dst_key` | no | — | SSH private key for the destination platform, only required when the `dst` platform is `gitee` |
| `src_endpoint` | no | — | Self-hosted endpoint for the source platform, only used when the `src` platform is `gitlab` |
| `dst_endpoint` | no | — | Self-hosted endpoint for the destination platform, only used when the `dst` platform is `gitlab` |
| `repo-names` | no | (all repos) | Comma-separated repo names to mirror |
| `force` | no | `true` | Force-push mirrored refs, overwriting divergent history on the destination |
| `workers` | no | `8` | Number of repos to mirror concurrently |

## Outputs

| Name | Description |
|---|---|
| `mirrored` | Number of repos actually mirrored (excludes repos skipped because they were already up to date) |
| `skipped` | Number of repos skipped because they were already up to date |
| `failed` | Number of repos that failed to mirror |
| `total` | Number of repos attempted |

## Usage

```yaml
- name: Mirror to Gitee
  uses: farfarfun-action/mirror-repo@v1
  with:
    src: github/${{ github.repository_owner }}
    dst: gitee/farfarfun-skills
    dst_key: ${{ secrets.GITEE_RSA_PRIVATE_KEY }}
    dst_token: ${{ secrets.GITEE_TOKEN }}
    # Optional: skip auto-discovery by passing repo names directly,
    # e.g. from farfarfun-action/sync-forks's `all-repo-names` output.
    repo-names: ${{ steps.sync.outputs.all-repo-names }}
```
