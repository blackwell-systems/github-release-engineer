# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [v0.1.4] - 2026-03-06

### Added

- YAML frontmatter: `name`, `description`, `disable-model-invocation: true`, and `argument-hint` — migrates to the Claude Code skills spec and prevents Claude from auto-invoking the release workflow
- `$ARGUMENTS` placeholder in the Arguments section so Claude receives the exact invocation string rather than relying on auto-appended arguments

### Changed

- README framing updated to lead with the [Agent Skills](https://agentskills.io) open standard — skill is no longer described as Claude Code-specific
- Install method changed from `cp` to `ln -s` — symlink keeps the global skill in sync with the repo automatically
- Install target updated from `.claude/commands/release.md` to `.claude/skills/release/SKILL.md` — adopts the current skills directory format

### Removed

- Duplicate `release.md` at the repo root (canonical source is `prompts/release-skill.md`)

## [v0.1.3] - 2026-03-01

### Added

- `--allow-ahead` flag: ahead-of-remote is now a hard abort by default; pass `--allow-ahead` to override
- `--ci-discovery-timeout` argument to control how long CI run discovery polls before falling back
- Step 4: show full copyable SHA when confirming commit to tag; allow tagging a non-HEAD commit by accepting a user-provided SHA
- Step 10: derived boolean table (`isDraft`/`isPrerelease` per flag combination) for explicit release state verification

### Changed

- CI discovery polling is now explicit: poll every `--poll-interval` seconds for up to `--ci-discovery-timeout` seconds, then fall back to a filterless run list matched on `headSha`
- Asset stability counter explicitly resets on count change between polls
- Checksum source specified: attached `.sha256`/`.checksums.txt` release assets only — do not download and hash locally

## [v0.1.2] - 2026-03-01

### Added

- State capture table at the top of the skill — six variables captured once and referenced throughout all steps
- `--retries` argument added to the Arguments section
- Companion skill payload now includes checksums and `tag_sha`
- Changelog absence noted in final report warnings

### Changed

- CI run selection uses `headSha` matching instead of `--branch refs/tags/...` for more reliable matching across workflow configurations
- Tag remote existence check uses exact ref with `^{}` annotation handling
- Default branch normalized from `refs/remotes/origin/main` to `main`
- Up-to-date check uses `git rev-list --left-right --count`, requiring `0\t0` output
- Release existence check handles non-zero exit from `gh release view`
- Retag guardrail explicitly covers the prerelease-but-published case

## [v0.1.1] - 2026-03-01

### Added

- Pre-flight checks for `gh auth status`, remote reachability via `gh repo view`, and in-progress git operations (`MERGE_HEAD`, `CHERRY_PICK_HEAD`, `REVERT_HEAD`, rebase directories)
- Version resolution with explicit precedence order covering Go, Rust, Python, Java, JS, and Gradle manifests
- Semver normalization rules: strip and re-add `v` prefix, no build metadata allowed in tags
- Tag safety checks before creating a tag: local existence, remote existence, commit confirmation
- CI watch handling for multiple concurrent runs and the "no run found" case
- Asset polling waits for a stable count across two consecutive polls
- Error handling policy table

### Changed

- Retagging blocked if a published release already exists
- Release creation responsibility explicitly defined
- `--pre-release` vs `--draft` clarified as distinct flags with different GitHub field effects

## [v0.1.0] - 2026-03-01

### Added

- Initial release
- Linear 10-step pipeline: pre-flight, version, changelog, tag, push, watch CI, diagnose/fix, wait for artifacts, verify, report
- Gated steps with explicit stop conditions
- CI fix-and-retry loop with configurable retry limit
- Artifact polling with size > 0 check
- No force-push by default
- Companion skill handoff at Step 10
