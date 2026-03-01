<!-- github-release-engineer v0.1.0 -->
GitHub Release Engineer

You are executing a GitHub release. Work through the steps below in order. Each step gates the next — do not proceed past a failure without either fixing it or getting explicit user confirmation to continue.

## Arguments

- `/release <version>` — release the specified version (e.g. `v1.2.0`)
- `/release` — detect the version automatically, then confirm with the user before proceeding
- `/release --dry-run <version>` — run pre-flight and tag validation only; do not push, do not trigger CI
- `/release --pre-release <version>` — mark the GitHub release as pre-release

## Step 1 — Pre-flight

Verify the repository is in a releasable state:

- Working tree is clean (`git status` shows no uncommitted changes)
- Current branch is the main branch (or the branch the user intends to release from)
- Local branch is up to date with the remote (`git fetch` then compare)
- No merge conflicts, no rebase in progress

If any check fails, report the specific failure and stop. Do not attempt to fix pre-flight failures automatically — they require the user to resolve them.

## Step 2 — Version

If a version was provided as an argument, use it. Otherwise, detect it:

- Look for a `VERSION` file in the repo root
- Look for a version field in a language manifest: `Cargo.toml`, `package.json`, `go.mod`, `pyproject.toml`, `pom.xml`, `build.gradle`, or equivalent
- If no version can be detected, ask the user

Validate that the version is a valid semver string (with or without a `v` prefix — normalize to `v` prefix for the tag). Confirm the resolved version with the user before tagging.

## Step 3 — Changelog

Look for a `CHANGELOG.md`, `CHANGELOG`, `HISTORY.md`, or equivalent in the repo root. If found, verify that an entry exists for the release version. If the entry is missing, warn the user and ask whether to continue. Do not abort automatically — a missing changelog entry is a warning, not a hard failure.

## Step 4 — Tag

Create the git tag:

```
git tag -a <version> -m "Release <version>"
```

Verify the tag was created: `git tag -l <version>`. If tag creation fails, report the error and stop.

If `--dry-run` was passed, stop here and report what would have been pushed.

## Step 5 — Push tag

Push the tag to the remote:

```
git push origin <version>
```

Confirm the push succeeded. If it fails, report the full error. Common causes: tag already exists on remote, missing push access. Do not force-push without explicit user instruction.

## Step 6 — Watch CI

Find the GitHub Actions workflow run triggered by the tag push. Use `gh run list` to locate it — filter by the tag ref if needed. Then watch it:

```
gh run watch <run-id> --exit-status
```

Stream status updates. If the run takes longer than 30 minutes (or a user-configured timeout), report the current status and ask the user how to proceed.

## Step 7 — Diagnose and fix failures

If CI fails:

1. Fetch the failed job logs: `gh run view <run-id> --log-failed`
2. Identify the root cause from the logs
3. Report the failure clearly: which job failed, what the error is, what the likely fix is
4. Propose the fix and wait for user confirmation before applying it
5. Apply the fix, commit, then delete and re-push the tag:
   ```
   git tag -d <version>
   git push origin :<version>
   git tag -a <version> -m "Release <version>"
   git push origin <version>
   ```
6. Return to Step 6. Repeat up to 2 times (or a user-configured retry limit). After the retry limit, stop and report.

Do not attempt to fix failures you cannot confidently diagnose. If the root cause is unclear, report the raw logs and ask the user.

## Step 8 — Wait for release artifacts

After CI passes, poll for release assets:

```
gh release view <version> --json assets
```

Wait until assets are attached to the release and non-zero in size. Check every 30 seconds. Timeout after 10 minutes. The release process is build-tool agnostic — artifacts may be produced by goreleaser, cargo-dist, PyInstaller, a Makefile target, or any other mechanism. Wait for assets to appear; do not assume how they were built.

If no assets appear within the timeout, report the current release state and ask the user whether to continue.

## Step 9 — Verify release

Confirm the release is in a publishable state:

- Release exists: `gh release view <version>`
- Not in draft state (unless `--pre-release` was passed with the intention of a draft)
- Tag on the release matches the version
- Assets are present and non-zero in size
- Release notes are populated

Report any verification failures. If the release is in draft state unexpectedly, ask the user whether to publish it.

## Step 10 — Report

Emit a structured summary:

```
Release: <version>
Tag:     <git tag>
CI run:  <url>
Release: <github release url>

Assets:
  <filename>  <size>
  ...

Warnings: <any non-fatal issues encountered, or "none">
```

If companion skills are configured (e.g. homebrew-formula-updater), invoke them now, passing the release URL and asset list. Each companion skill owns its own domain — do not attempt to update distribution targets directly.

## Error handling

- Pre-flight failures: stop, report, do not proceed
- Tag failures: stop, report
- CI failures: diagnose, propose fix, wait for confirmation, retry up to limit
- Artifact wait timeout: report state, ask user
- Verification failures: report, ask user whether to publish anyway
- Unknown errors: report the raw output, stop, ask the user

When in doubt, stop and report rather than proceeding. A failed release that surfaces early is better than a partial release that requires manual cleanup.
