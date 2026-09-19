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
| `state-file` | no | — | Path (relative to the workspace) to a JSON file mapping repo name to last-synced source sha; read at the start, rewritten at the end. Persisting it across runs (e.g. committing it back to the calling repo) is the caller's job |
| `incremental` | no | `false` | When `true` and `state-file` is set, skip repos whose current source sha still matches `state-file` without ever querying the destination. `false` always checks the destination for real and refreshes `state-file` from the confirmed results |

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

### Incremental runs against a throttled destination

If the destination platform (e.g. Gitee) rate-limits or WAF-blocks under
load, run this action frequently with `incremental: true`, and periodically
(e.g. daily) with `incremental: false` to self-heal any drift. The caller
owns checking out and committing `state-file` back to its own repo — this
action only reads and rewrites the local file:

```yaml
- uses: actions/checkout@v4
- uses: farfarfun-action/mirror-repo@v1
  with:
    src: github/my-org
    dst: gitee/my-org
    dst_key: ${{ secrets.GITEE_RSA_PRIVATE_KEY }}
    dst_token: ${{ secrets.GITEE_TOKEN }}
    state-file: .mirror-state/gitee.json
    incremental: 'true'   # 'false' on the periodic full-sync schedule
    workers: '2'          # keep low on the full-sync schedule
- name: Persist mirror state
  if: always()
  run: |
    git config user.name 'github-actions[bot]'
    git config user.email 'github-actions[bot]@users.noreply.github.com'
    git add .mirror-state/gitee.json
    git diff --cached --quiet || git commit -m 'chore: update Gitee mirror state [skip ci]'
    git push
```
