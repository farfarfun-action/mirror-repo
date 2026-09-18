# mirror-to-gitee

Mirror all (or a given list of) repos in a GitHub organization to a Gitee
organization, via [`Yikun/hub-mirror-action`](https://github.com/Yikun/hub-mirror-action).

If `repo-names` is not supplied, the action lists every repo in `github-org`
itself (requires `github-token`).

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `github-org` | yes | — | Source GitHub organization |
| `gitee-org` | yes | — | Destination Gitee organization |
| `gitee-key` | yes | — | Gitee RSA private key |
| `gitee-token` | yes | — | Gitee access token |
| `repo-names` | no | (all repos) | Comma-separated repo names to mirror |
| `github-token` | no | — | Token used to list repos when `repo-names` is not provided |

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
