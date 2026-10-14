# mirror-repo

Mirror all (or a given list of) repos from one hosting platform's org to
another's — GitHub, Gitee, GitLab or GitCode, in any src/dst combination.

If `repo-names` is not supplied, the action lists every repo in `src` itself
(requires `src_token`).

## Built on funmirror

This action is a thin GitHub Actions wrapper: it parses `src`/`dst`, stages
any SSH key the destination needs, `pip install`s
[`farfarfun/funmirror`](https://github.com/farfarfun/funmirror), and shells
out to its `funmirror mirror` CLI — no third-party mirror action dependency.
All of the actual mirroring logic lives in that package:

- **Two-phase pipeline** — a cheap, read-only *detect* phase checks every
  repo's commit id at high concurrency (`detect-workers`); only repos that
  actually differ move on to the heavier *sync* phase (`workers`, kept much
  lower to avoid overwhelming a rate-limited destination like Gitee).
- **Skip-if-unchanged** — a repo whose source and destination default branch
  already share the same latest commit is skipped without cloning anything.
- **Incremental + full sync** — see `incremental` below; full sync
  self-heals any drift an incremental run left behind.
- **Release mirroring** — opt-in via `sync-releases`, covering the files
  attached to each release, not just git refs.

<img src="docs/architecture.png" alt="funmirror / mirror-repo pipeline architecture" width="100%">

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
| `workers` | no | `8` | Number of repos to mirror concurrently (clone+push phase) |
| `detect-workers` | no | `max(workers*4, 16)` | Concurrency for the read-only commit-id detection phase that runs before clone+push. Keep `workers` low to protect a rate-limited destination while detection still runs fast, since it never touches git |
| `state-file` | no | — | Path (relative to the workspace) to a JSON file, namespaced by `<src>::<dst>`, mapping repo name to `{src_sha, dst_sha}`; read at the start, rewritten at the end. Persisting it across runs (e.g. committing it back to the calling repo) is the caller's job |
| `incremental` | no | `false` | When `true` and `state-file` is set, only the source platform's commit sha is queried; a repo whose sha still matches `state-file` is skipped without ever querying the destination. `false` (full sync) queries both source and destination for every repo, ignores `state-file` for the decision, and refreshes it from the confirmed results — self-healing any drift |
| `sync-releases` | no | `false` | Also mirror releases and the files attached to them, in a pass that runs after the git push. Needs `dst_token`. Currently `github` → `gitee` only. See below |
| `release-limit` | no | (all) | Only mirror the N most recent releases per repo |

## Outputs

| Name | Description |
|---|---|
| `mirrored` | Number of repos actually mirrored (excludes repos skipped because they were already up to date) |
| `skipped` | Number of repos skipped because they were already up to date |
| `failed` | Number of repos that failed to mirror |
| `total` | Number of repos attempted |

## Usage

> **Pin an exact tag, not `@v1`.** The floating `v1` tag is frozen at `30c4b5c`,
> one commit before the `ssh-keyscan` fix in `v1.1.0` — on that version a single
> blip reaching `gitee.com:22` kills the whole run with an `##[error]` and no
> explanation. `v1` was left where it is on purpose rather than moved, so
> anything still on it keeps getting the old behaviour. Use `@v1.1.0`.

```yaml
- name: Mirror to Gitee
  uses: farfarfun-action/mirror-repo@v1.1.0
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
- uses: farfarfun-action/mirror-repo@v1.1.0
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

### Mirroring releases and their files

A git mirror copies refs. It does not copy releases, and it does not copy
the binaries attached to them — which for a tool distributed as a prebuilt
binary is the only part users actually need. `sync-releases: 'true'` adds a
pass after the push that creates any missing release on the destination (by
tag) and uploads the attachments it's missing:

```yaml
- uses: farfarfun-action/mirror-repo@v1.1.0
  with:
    src: github/${{ github.repository_owner }}
    dst: gitee/${{ github.repository_owner }}
    src_token: ${{ secrets.GITHUB_TOKEN }}
    dst_key: ${{ secrets.GITEE_RSA_PRIVATE_KEY }}
    dst_token: ${{ secrets.GITEE_TOKEN }}
    repo-names: ${{ github.event.repository.name }}
    sync-releases: 'true'
    release-limit: '5'
```

`dst_token` is not optional here: the SSH key covers git, not the REST API
that creates releases and uploads files.

Both passes are idempotent and diff-based — a release already there is
matched by tag and left alone, an attachment already there is matched by
name and only re-uploaded if its size disagrees. Re-running costs API calls,
not bandwidth.

Two things to know before turning it on:

- **Every selected repo pays a release listing on both sides, every run,
  including the unchanged ones.** Publishing a release moves a tag, not the
  default branch, so a repo whose commits are identical can still be missing
  its newest release — skipping those on the commit sha alone would miss
  exactly what this switch is for. Those repos still skip the clone+push;
  only the release diff runs.
- **`release-limit` is worth setting on the first run.** Without it, a repo
  with a long history of large binaries moves all of it at once.
