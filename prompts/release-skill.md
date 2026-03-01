<!-- github-release-engineer v0.1.1 -->
GitHub Release Engineer

You are executing a GitHub release. Work through the steps below in order. Each step gates the next — do not proceed past a failure without either fixing it or getting explicit user confirmation to continue.

## Arguments

- `/release <version>` — release the specified version (e.g. `v1.2.0`)
- `/release` — detect the version automatically, then confirm with the user before proceeding
- `/release --dry-run <version>` — run pre-flight and tag validation only; do not push, do not trigger CI
- `/release --pre-release <version>` — mark the GitHub release as pre-release (sets `prerelease=true`, `draft=false`)
- `/release --draft <version>` — create the release as a draft (sets `draft=true`)

Timeouts are configurable via flags: `--ci-timeout <minutes>` (default 30), `--asset-timeout <minutes>` (default 10), `--poll-interval <seconds>` (default 30).

## Step 1 — Pre-flight

Verify the repository is in a releasable state. Run all checks before reporting failures.

**Authentication and remote:**
- Verify GitHub CLI is authenticated: `gh auth status`. Abort if not.
- Determine the remote: `git remote get-url origin`. Abort if missing.
- Verify the remote points to a reachable GitHub repo: `gh repo view`. Abort if not.

**Working tree:**
- No uncommitted changes: `git status --porcelain` must be empty
- No merge, cherry-pick, or revert in progress: check for `.git/MERGE_HEAD`, `.git/CHERRY_PICK_HEAD`, `.git/REVERT_HEAD`
- No rebase in progress: check for `.git/rebase-merge/` or `.git/rebase-apply/`

**Branch:**
- Determine the default branch: `git symbolic-ref refs/remotes/origin/HEAD`. If unset, ask the user which branch to release from.
- Confirm the current branch is the release branch, or ask the user to confirm if it is not.
- Local branch is up to date with the remote: `git fetch origin` then compare.

If any check fails, report the specific failure and stop. Do not attempt to fix pre-flight failures automatically.

## Step 2 — Version

**If a version was provided as an argument**, use it.

**Otherwise, detect it in this order:**
1. `VERSION` file in the repo root
2. Language manifest: `Cargo.toml` (`version =`), `package.json` (`"version":`), `pyproject.toml` (`version =`), `pom.xml` (`<version>`), `build.gradle` (`version =`), or equivalent
3. If no version can be detected, ask the user

**Normalization:**
- Strip a leading `v` prefix if present, then re-add it: all tags are `vX.Y.Z`
- Validate strict semver: `MAJOR.MINOR.PATCH` with optional pre-release identifiers (`-alpha.1`)
- Do not allow build metadata (`+...`) in tags
- If multiple manifests exist, use the first match in the precedence order above and note which file was used

Confirm the resolved version with the user before proceeding.

## Step 3 — Changelog

Look for `CHANGELOG.md`, `CHANGELOG`, `HISTORY.md`, or equivalent in the repo root. If found, search for an entry matching the version. Accept common formats: `## v1.2.3`, `## 1.2.3`, `## [1.2.3]`, `## [v1.2.3]`. Normalize the version when searching (strip `v` prefix for comparison).

If the entry is missing, warn the user and ask whether to continue. Do not abort automatically — a missing changelog entry is a warning, not a hard failure. If no changelog file exists, skip silently.

## Step 4 — Tag safety checks

Before creating the tag:

- **Tag must not exist locally**: `git tag -l <version>` must return empty. If it exists, ask the user: delete and recreate, or abort?
- **Tag must not exist on remote**: `git ls-remote --tags origin <version>` must return empty. If it exists, do not delete it automatically — report it and ask the user how to proceed.
- **Confirm the commit to tag**: default is `HEAD`. Show the commit hash and message, and ask the user to confirm.

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

Find the GitHub Actions workflow run triggered by the tag push:

```
gh run list --event push --branch refs/tags/<version>
```

If multiple runs are found, select the most recent one on `refs/tags/<version>`. If no run is found within 60 seconds of the push, report the absence and ask the user whether to wait longer or skip CI monitoring. If CI is known to trigger on release creation rather than tag push, note this and proceed to Step 8.

Watch the selected run:

```
gh run watch <run-id> --exit-status
```

Stream status updates. Time out after `--ci-timeout` minutes and report the current status, then ask the user how to proceed.

## Step 8 — Diagnose and fix failures

If CI fails:

1. Fetch the failed job logs: `gh run view <run-id> --log-failed`
2. Identify the root cause from the logs
3. Report clearly: which job failed, what the error is, what the likely fix is
4. Propose the fix and wait for user confirmation before applying it
5. Apply the fix and commit

**On retagging:** Before deleting and re-pushing the tag, verify the tag is not already associated with a published (non-draft) GitHub release: `gh release view <version> --json isDraft`. If a published release exists, do not retag — report this and ask the user whether to release a patch version instead. If no release exists or it is a draft, proceed:

```
git tag -d <version>
git push origin :<version>
git tag -a <version> -m "Release <version>"
git push origin <version>
```

Return to Step 7. Repeat up to `--retries` times (default 2). After the retry limit, stop and report.

Do not attempt to fix failures you cannot confidently diagnose. If the root cause is unclear, report the raw logs and ask the user.

## Step 9 — Wait for release artifacts

After CI passes, determine whether a GitHub release was created automatically (by goreleaser, cargo-dist, a CI step, or another mechanism):

```
gh release view <version> --json isDraft,assets
```

If no release exists, ask the user: should one be created now (`gh release create`), or is this expected (e.g., manual upload)?

Once the release exists, poll for assets every `--poll-interval` seconds:

```
gh release view <version> --json assets
```

Wait until:
- At least one asset is attached
- All assets are non-zero in size
- Asset count is stable across two consecutive polls (avoids mid-upload state)

Time out after `--asset-timeout` minutes and report the current release state, then ask the user whether to continue.

## Step 10 — Verify release

Confirm the release is in the expected state:

- Release exists: `gh release view <version>`
- `isDraft` is false (unless `--draft` was passed)
- `isPrerelease` matches the `--pre-release` flag
- Tag on the release matches the version
- Assets are present and non-zero in size
- Release notes are populated

Report any verification failures. If the release is in draft state unexpectedly, ask the user whether to publish it.

## Step 11 — Report

Emit a structured summary:

```
Release: <version>
Tag:     <git tag> (<commit hash>)
CI run:  <url>
Release: <github release url>

Assets:
  <filename>  <size>  <url>
  ...

Warnings: <any non-fatal issues encountered, or "none">
```

If companion skills are configured (e.g. `homebrew-formula-updater`), invoke them now. Pass: release URL, tag, and the full asset list with filenames, URLs, and sizes. Companion skill failures are reported as warnings in the summary — they do not retroactively fail the release.

## Error handling policy

| Failure | Policy |
|---|---|
| Pre-flight failure | Stop, report, do not proceed |
| Version unresolvable | Ask user |
| Changelog entry missing | Warn, ask user |
| Tag exists locally | Ask user: delete or abort |
| Tag exists on remote | Report, ask user |
| Push fails | Stop, report |
| No CI run found | Report, ask user |
| CI failure | Diagnose, propose fix, confirm, retry up to limit |
| Published release exists on retag | Report, suggest patch version |
| No release after CI | Ask user |
| Asset wait timeout | Report state, ask user |
| Verification failure | Report, ask user |
| Unknown error | Report raw output, stop, ask user |

When in doubt, stop and report rather than proceeding. A failed release that surfaces early is better than a partial release that requires manual cleanup.
