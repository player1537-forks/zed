---
name: fork-release
description: Trigger and monitor a fork-release build for the player1537-forks/zed fork. Use this when the user asks to "do a release", "make a new release", "start a fork-release", or wants to retry a failed fork-release build.
---

# Fork Release Build

Trigger and trace a release build of the `player1537-forks/zed` fork. This drives the
`.github/workflows/fork-release.yml` workflow, which builds a macOS app and two Linux
remote servers, then publishes a GitHub Release with a `vX.Y.0-fork` tag.

## When to use

Use this when the user wants to cut a new fork release (or retry one that failed before
publishing). The workflow is manually dispatched, not triggered by pushing a tag — you
trigger it yourself via `gh workflow run`.

## Key facts

- **Repo**: `player1537-forks/zed` (always pass `--repo player1537-forks/zed` to `gh`; the local checkout may be a different remote or a worktree).
- **Workflow name**: `Fork Release Build` (file `.github/workflows/fork-release.yml`, workflow ID `326303420`).
- **Inputs**: `tag_name` (required) and `build_ref` (required, normally `main`).
- **The workflow creates the tag itself** via `softprops/action-gh-release@v2` in the `create-release` job. Do **not** push or create the tag manually.
- **Release channel**: the workflow forces `crates/zed/RELEASE_CHANNEL` to `dev` before building (fork builds are always Zed Dev).
- **Jobs**: `build-macos`, `build-linux-remote-server-x86_64`, `build-linux-remote-server-aarch64`, then `create-release` (which needs all three).

## Determining the next version

Fork-release tags are exactly `vX.Y.0-fork` (no extra suffix). Ignore other tags like
`v0.3.0-fork-preview`, `v0.3.0-fork-dev`, or `v0.2.0-fork-titlebar-color` — those come
from different workflows.

List tags and find the highest plain `-fork` version:

```
gh api repos/player1537-forks/zed/tags --jq '.[].name'
```

Take the highest tag matching `^v\d+\.\d+\.0-fork$` and increment the **minor** component
(the `Y` in `vX.Y.0-fork`). For example, from `v0.11.0-fork` the next is `v0.12.0-fork`.

## Procedure

### 1. Trigger the workflow

```
gh workflow run "Fork Release Build" --repo player1537-forks/zed \
  -f tag_name=<next-tag> -f build_ref=main
```

### 2. Find the run ID

The dispatch is asynchronous — poll the workflow's run list until a fresh run appears:

```
gh run list --repo player1537-forks/zed --workflow="fork-release.yml" --limit 1 \
  --json databaseId,status,url,createdAt
```

### 3. Monitor progress

Do **not** use `gh run watch --exit-status` — its default 300s timeout is far shorter than
the multi-hour Zed build, and it will time out. Also note `gh run view --log` only works on
**completed** runs. Instead poll the Jobs API for intermediate per-job status:

```
gh api repos/player1537-forks/zed/actions/runs/<run-id>/jobs \
  --jq '.jobs[] | "\(.name) | status: \(.status) | conclusion: \(.conclusion // "—")"'
```

To block until the whole run finishes, poll the run status (handles the queued → in_progress
→ completed transitions):

```
while true; do
  status=$(gh run view <run-id> --repo player1537-forks/zed \
    --json status,conclusion --jq '"\(.status) \(.conclusion // "—")"')
  echo "$(date '+%H:%M:%S') — $status"
  echo "$status" | grep -q "completed" && break
  sleep 120
done
```

Approximate job durations (varies with cache): `build-linux-remote-server-aarch64` ~15–25 min,
`build-linux-remote-server-x86_64` ~20–30 min, `build-macos` ~50–75 min, `create-release` a few
seconds.

### 4. Confirm success

Once the run is `completed`, check every job concluded `success`:

```
gh api repos/player1537-forks/zed/actions/runs/<run-id>/jobs \
  --jq '.jobs[] | "\(.name) | \(.conclusion)"'
```

On success, the release is live at
`https://github.com/player1537-forks/zed/releases/tag/<tag>`. Report the run URL and release
URL to the user.

## Retrying a failed build

If a build job fails (e.g. a spurious host error) before `create-release` runs, the
`create-release` job is `skipped` and **no tag is created**. In that case, re-trigger with the
**same** `tag_name` — do not bump the version, or you'll leave a gap in the tag sequence.

Confirm whether the tag was created before deciding:

```
gh api repos/player1537-forks/zed/tags --jq '.[].name' | grep -x '<tag>'
```

Only bump to the next version once a run with `<tag>` has fully succeeded.

## Gotchas

- **Always pass `--repo player1537-forks/zed`** to `gh`. The local working directory is a
  Zed checkout, but releases are built and published from the GitHub fork.
- **`gh run watch` is a trap** for this workflow. Use the Jobs API or `gh run view --json`.
- **`gh run view --log` on a running job fails** — only inspect logs after the run completes.
- **Don't create the tag yourself.** `create-release` creates it; a manual tag would make the
  release step fail or point at the wrong commit.
