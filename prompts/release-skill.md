<!-- github-release-engineer v0.1.4 -->
GitHub Release Engineer

You are executing a GitHub release. Work through the steps below in order. Each step gates the next — do not proceed past a failure without either fixing it or getting explicit user confirmation to continue.

**Background execution:** All long-running blocking commands (`gh run watch`, polling loops, asset wait loops) must use `run_in_background: true` and be awaited via `TaskOutput`. Never run watch or poll commands as foreground blocking calls — the user should not have to manually background anything.

## Arguments

- `/release <version>` — release the specified version (e.g. `v1.2.0`)
- `/release` — detect the version automatically, then confirm with the user before proceeding
- `/release --dry-run <version>` — run pre-flight and tag validation only; do not push, do not trigger CI
- `/release --pre-release <version>` — mark the GitHub release as pre-release (`prerelease=true`, `draft=false`)
- `/release --draft <version>` — create the release as a draft (`draft=true`, `prerelease=false`)
- `/release --allow-ahead <version>` — allow releasing when local branch is ahead of remote (default: abort)

Timeouts are configurable: `--ci-timeout <minutes>` (default 30), `--ci-discovery-timeout <seconds>` (default 60), `--asset-timeout <minutes>` (default 10), `--poll-interval <seconds>` (default 30), `--retries <n>` (default 2).

## State

Capture the following as you progress through the steps. Reference these values in later steps rather than re-deriving them:

| Variable | Set in step | Value |
|---|---|---|
| `remote_url` | Step 1 | `git remote get-url origin` |
| `release_branch` | Step 1 | default branch, normalized |
| `version` | Step 2 | normalized semver with `v` prefix |
| `tag_sha` | Step 4 | commit SHA being tagged |
| `ci_run_id` | Step 7 | GitHub Actions run ID |
| `release_url` | Step 9 | GitHub release URL |

## Step 1 — Pre-flight

Verify the repository is in a releasable state. Run all checks before reporting failures.

**Authentication and remote:**
- Verify GitHub CLI is authenticated: `gh auth status`. Abort if not.
- Determine the remote URL: `git remote get-url origin`. Abort if missing. Store as `remote_url`.
- Verify the remote is reachable and `gh` is scoped to this repo: run `gh repo view` from inside the repo working directory. Abort if not.

**Working tree:**
- No uncommitted changes: `git status --porcelain` must be empty
- No merge, cherry-pick, or revert in progress: check for `.git/MERGE_HEAD`, `.git/CHERRY_PICK_HEAD`, `.git/REVERT_HEAD`
- No rebase in progress: check for `.git/rebase-merge/` or `.git/rebase-apply/`

**Branch:**
- Determine the default branch: `git symbolic-ref refs/remotes/origin/HEAD`. Extract the last path component (e.g. `refs/remotes/origin/main` → `main`). Store as `release_branch`. If the command fails or is unset, ask the user which branch to release from.
- Confirm the current branch matches `release_branch`, or ask the user to confirm if it does not.
- Fetch and verify up to date: `git fetch origin`, then `git rev-list --left-right --count HEAD...@{u}`. The output is `<ahead>\t<behind>`.
  - `behind > 0`: abort — local branch is behind remote.
  - `ahead > 0` and `--allow-ahead` not passed: abort — local branch has unpublished commits. Releasing from unpublished commits means the release tag cannot be reproduced from the remote state. Pass `--allow-ahead` explicitly to override.
  - `ahead > 0` and `--allow-ahead` passed: warn and note in final report warnings.
  - `0\t0`: proceed.

If any check fails, report the specific failure and stop. Do not attempt to fix pre-flight failures automatically.

## Step 2 — Version

**If a version was provided as an argument**, use it.

**Otherwise, detect it in this order** (first match wins):
1. `VERSION` file in the repo root
2. `Cargo.toml` — `version =` field under `[package]`
3. `package.json` — `"version":` field at the top level
4. `pyproject.toml` — `version =` field under `[tool.poetry]` or `[project]`
5. `pom.xml` — first `<version>` element under `<project>`
6. `build.gradle` or `build.gradle.kts` — `version =` field

Only these files are checked. If none match and no version was provided, ask the user. Do not scan other files.

**Normalization:**
- Strip a leading `v` prefix if present, then re-add it: all tags are `vX.Y.Z`
- Validate strict semver: `MAJOR.MINOR.PATCH` with optional pre-release identifiers (`-alpha.1`, `-rc.2`)
- Do not allow build metadata (`+...`) in tags
- If multiple manifests exist, use the first match in the precedence order above; note which file was used

Confirm the resolved version with the user before proceeding.

## Step 3 — Changelog

Look for `CHANGELOG.md`, `CHANGELOG`, `HISTORY.md`, or equivalent in the repo root. If no changelog file exists, note "No changelog file found" in the final report warnings and continue.

If found, search for an entry matching the version. Accept common formats: `## v1.2.3`, `## 1.2.3`, `## [1.2.3]`, `## [v1.2.3]`. Normalize version when searching (compare without `v` prefix).

If the entry is missing, warn the user and ask whether to continue. A missing entry is a warning, not a hard failure.

## Step 4 — Tag safety checks

Before creating the tag:

- **Tag must not exist locally**: `git tag -l <version>` must return empty. If it exists, ask the user: delete and recreate, or abort?
- **Tag must not exist on remote**: `git ls-remote --tags origin "refs/tags/<version>"`. Treat any output (including `refs/tags/<version>^{}` for annotated tags) as "exists." If the tag exists on the remote, report it and ask the user how to proceed — do not delete it automatically.
- **Confirm commit to tag**: default is `HEAD`. Show the full commit SHA (copyable) and commit message. Ask the user to confirm this is the correct commit. If the user wishes to tag a different commit, accept a SHA as input. Store the confirmed SHA as `tag_sha`.

## Step 5 — Tag

Create the tag:

```
git tag -a <version> -m "Release <version>"
```

Verify the tag was created: `git tag -l <version>`. If tag creation fails, report the error and stop.

If `--dry-run` was passed, stop here and report what would have been pushed.

## Step 6 — Push tag

Push the tag to the remote:

```
git push origin <version>
```

Confirm the push succeeded. If it fails, report the full error and stop. Do not force-push without explicit user instruction.

## Step 7 — Watch CI

Locate the GitHub Actions run triggered by the tag push. Poll every `--poll-interval` seconds for up to `--ci-discovery-timeout` seconds total. On each poll:

```
gh run list --event push --json databaseId,headSha,headBranch,displayTitle,createdAt,url
```

Select the run whose `headSha` matches `tag_sha`. If multiple runs match, select the most recent by `createdAt`. If no match is found after `--ci-discovery-timeout` seconds, retry without the event filter (some tag-triggered workflows may appear under a different event type):

```
gh run list --json databaseId,headSha,headBranch,displayTitle,createdAt,url
```

Again match on `headSha == tag_sha`. If still no match, report the absence and ask the user whether to wait longer, skip CI monitoring, or abort.

Note: some projects trigger CI on `release` creation rather than tag push. If no run is found and release-triggered CI is suspected, report this and proceed to Step 9.

Store the matched run ID as `ci_run_id`. Watch the run using `run_in_background: true` so it does not block the foreground session:

```
gh run watch <ci_run_id> --exit-status
```

Launch this as a background Bash task. Wait for the task to complete via TaskOutput. Do not run it as a foreground blocking command — the user should not need to manually background it. Time out after `--ci-timeout` minutes, report the current status, and ask the user how to proceed.

## Step 8 — Diagnose and fix failures

If CI fails:

1. Fetch the failed job logs: `gh run view <ci_run_id> --log-failed`
2. Identify the root cause from the logs
3. Report clearly: which job failed, what the error is, what the likely fix is
4. Propose the fix and wait for user confirmation before applying it
5. Apply the fix and commit

**Retag safety check:** Before retagging, check whether a non-draft release already exists:

```
gh release view <version> --json isDraft
```

If the command returns non-zero (release not found), proceed with retagging. If the release exists and `isDraft` is `false` — regardless of prerelease status, a published prerelease still counts as published — do not retag. Report this and ask the user whether to release a patch version instead.

If retagging is safe:

```
git tag -d <version>
git push origin :"refs/tags/<version>"
git tag -a <version> -m "Release <version>"
git push origin <version>
```

Update `tag_sha` to the new commit SHA. Return to Step 7. Repeat up to `--retries` times. After the retry limit, stop and report.

Do not attempt to fix failures you cannot confidently diagnose. Report raw logs and ask the user.

## Step 9 — Wait for release artifacts

After CI passes, check whether a GitHub release was created automatically:

```
gh release view <version> --json isDraft,assets,url
```

If the command returns non-zero and indicates the release was not found, ask the user: create one now with `gh release create`, wait longer, or proceed without artifacts? Do not assume the absence of a release is a failure — some pipelines upload artifacts separately.

Once the release exists, store its URL as `release_url`. Poll for assets every `--poll-interval` seconds:

```
gh release view <version> --json assets
```

Wait until all of the following are true:
- At least one asset is attached
- All assets are non-zero in size (keep polling if any asset has size 0)
- Asset count is stable across two consecutive polls — if the count changes between polls, reset the stability counter and continue polling

Time out after `--asset-timeout` minutes, report the current release state, and ask the user whether to continue.

## Step 10 — Verify release

Determine expected release state from arguments:

| Flag passed | Expected `isDraft` | Expected `isPrerelease` |
|---|---|---|
| neither | `false` | `false` |
| `--pre-release` | `false` | `true` |
| `--draft` | `true` | `false` |

Confirm the release matches:

- Release exists: `gh release view <version>`
- `isDraft` matches expected value
- `isPrerelease` matches expected value
- Tag on the release matches `version`
- Assets are present and non-zero in size
- Release notes are populated

Report any verification failures. If the release is in draft state unexpectedly, ask the user whether to publish it.

## Step 11 — Report

Emit a structured summary:

```
Release: <version>
Tag:     <version> (<tag_sha>)
CI run:  <url>
Release: <release_url>

Assets:
  <filename>  <size>  <url>
  ...

Warnings: <any non-fatal issues, or "none">
```

Warnings must include any of: missing changelog entry, no changelog file found, ahead-of-remote (`--allow-ahead`), no CI run found (skipped), no assets found (skipped).

If companion skills are configured (e.g. `homebrew-formula-updater`), invoke them now. Pass: `release_url`, `version`, `tag_sha`, and the full asset list with filenames, download URLs, and sizes. For checksums: include any `.sha256`, `.sha512`, or `.checksums.txt` files attached as release assets; do not download and hash assets locally. Companion skill failures are reported as warnings in the summary — they do not retroactively fail the release.

## Error handling policy

| Failure | Policy |
|---|---|
| Pre-flight: auth/remote unreachable | Abort |
| Pre-flight: dirty working tree | Abort |
| Pre-flight: git operation in progress | Abort |
| Pre-flight: behind remote | Abort |
| Pre-flight: ahead of remote (no `--allow-ahead`) | Abort |
| Pre-flight: ahead of remote (`--allow-ahead`) | Warn, note in report |
| Version unresolvable | Ask user |
| Changelog entry missing | Warn, ask user |
| No changelog file | Note in warnings, continue |
| Tag exists locally | Ask user: delete or abort |
| Tag exists on remote | Report, ask user |
| Push fails | Abort |
| No CI run found within discovery timeout | Report, ask user |
| CI failure | Diagnose, propose fix, confirm, retry up to limit |
| Published release exists on retag | Report, suggest patch version |
| Release not found after CI | Ask user |
| Asset wait timeout | Report state, ask user |
| Verification failure | Report, ask user |
| Unknown error | Report raw output, abort, ask user |

When in doubt, stop and report rather than proceeding. A failed release that surfaces early is better than a partial release that requires manual cleanup.
