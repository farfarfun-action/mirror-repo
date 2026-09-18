# mirror-to-gitee

Mirror all (or a given list of) repos in a GitHub organization to a Gitee
organization. A thin wrapper around
[`farfarfun/funmirror`](https://github.com/farfarfun/funmirror), which
mirrors repos in parallel and skips any repo whose default branch already
has the same latest commit on both sides — no third-party mirror action
dependency.

If `repo-names` is not supplied, the action lists every repo in `github-org`
itself (requires `github-token`).

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `github-org` | yes | — | Source GitHub organization |
| `gitee-org` | yes | — | Destination Gitee organization |
| `gitee-key` | yes | — | Gitee SSH private key, used to push mirrored refs |
| `gitee-token` | yes | — | Gitee API access token, used to look up / create destination repos |
| `repo-names` | no | (all repos) | Comma-separated repo names to mirror |
| `github-token` | no | — | Token used to list repos when `repo-names` is not provided, and to clone private repos |
| `force` | no | `true` | Force-push mirrored refs, overwriting divergent history on the destination |
| `workers` | no | `8` | Number of repos to mirror concurrently |

## Outputs

| Name | Description |
|---|---|
| `mirrored` | Number of repos actually mirrored (excludes repos skipped because they were already up to date) |
| `total` | Number of repos attempted |

## Usage

```yaml
- name: Mirror to Gitee
  uses: farfarfun-action/mirror-to-gitee@v1
  with:
    github-org: ${{ github.repository_owner }}
    gitee-org: farfarfun-skills
    gitee-key: ${{ secrets.GITEE_RSA_PRIVATE_KEY }}
    gitee-token: ${{ secrets.GITEE_TOKEN }}
    # Optional: skip auto-discovery by passing repo names directly,
    # e.g. from farfarfun-action/sync-forks's `all-repo-names` output.
    repo-names: ${{ steps.sync.outputs.all-repo-names }}
```
