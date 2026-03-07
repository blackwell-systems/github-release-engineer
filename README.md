# github-release-engineer

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
[![Version](https://img.shields.io/github/v/release/blackwell-systems/github-release-engineer)](https://github.com/blackwell-systems/github-release-engineer/releases/latest)
[![Agent Skills](https://raw.githubusercontent.com/blackwell-systems/scout-and-wave/main/assets/badge-agentskills.svg)](https://agentskills.io)

An [Agent Skills](https://agentskills.io) skill that automates the full GitHub release process end-to-end — from tagging through CI, artifact production, and release verification.

## What It Is

`github-release-engineer` is a `/release` skill built on the [Agent Skills](https://agentskills.io) open standard. It handles everything on the GitHub side of a release: creating and pushing tags, monitoring CI, diagnosing and proposing fixes with user confirmation, waiting for release artifacts, and verifying the final release state.

It works in any Agent Skills-compatible tool. Install instructions below use Claude Code's skills directory as the reference implementation.

It lives in the blackwell-systems ecosystem alongside [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave), [agentic-cold-start-audit](https://github.com/blackwell-systems/agentic-cold-start-audit), and [homebrew-formula-updater](https://github.com/blackwell-systems/homebrew-formula-updater).

## What It Does Not Do

`github-release-engineer` is scoped to the GitHub side of the release lifecycle. It does not update distribution targets — Homebrew formulae, apt repositories, Docker Hub manifests, or any other downstream package registry. That work is explicitly out of scope.

Distribution updates are handled by companion skills such as [homebrew-formula-updater](https://github.com/blackwell-systems/homebrew-formula-updater). The release engineer calls them as a step. They own their domain; the release engineer owns GitHub.

## Process

The skill follows a fixed sequence. Each step gates the next:

1. **Pre-flight** — Verify GitHub CLI authentication, remote reachability, a clean working tree, no in-progress git operations, and that the local branch is not behind (or ahead without `--allow-ahead`) of the remote. Abort with a clear error if any check fails.
2. **Version** — Determine the release version: read from argument, `VERSION` file, or a language-specific manifest (`Cargo.toml`, `package.json`, `pyproject.toml`, `pom.xml`, `build.gradle`). Fall back to prompting the user. Validate strict semver and confirm with the user before proceeding.
3. **Changelog** — Look for a changelog file and search for an entry matching the version. Warn but do not abort if no changelog or entry is found; the user decides whether to continue.
4. **Tag safety checks** — Before creating the tag, confirm the tag does not already exist locally or on the remote, and confirm the exact commit SHA to tag with the user.
5. **Tag** — Create an annotated git tag (`git tag -a vX.Y.Z -m "Release vX.Y.Z"`). Verify the tag was created. Stop here if `--dry-run` was passed.
6. **Push tag** — Push the tag to the remote. Confirm the push succeeded. Do not force-push without explicit user instruction.
7. **Watch CI** — Locate the GitHub Actions run triggered by the tag push, matching on commit SHA. Poll until the run reaches a terminal state (success, failure, cancelled). Timeout after `--ci-timeout` minutes (default 30).
8. **Diagnose and fix failures** — If CI fails, fetch failed job logs, identify the root cause, propose a fix, apply it with user confirmation, commit, delete and re-push the tag, and re-enter the CI watch loop. Check for a published release before retagging. Repeat up to `--retries` times.
9. **Wait for release artifacts** — After CI passes, poll the GitHub release for attached assets every `--poll-interval` seconds. Wait until at least one asset is present, all assets are non-zero in size, and the asset count is stable across two consecutive polls. Timeout after `--asset-timeout` minutes (default 10). The skill is build-tool agnostic — it waits for assets regardless of how they were produced.
10. **Verify release** — Confirm the release is in the expected state (`isDraft`, `isPrerelease`) based on flags passed, the tag matches, assets are present and non-zero in size, and release notes are populated.
11. **Report** — Emit a structured summary: version, tag, CI run URL, release URL, artifact list with sizes, and any warnings encountered during the run. Invoke companion skills if configured.

## Composition with Companion Skills

The release engineer handles the GitHub side only. When the GitHub release is verified and the step report is emitted, the caller can invoke companion skills for distribution:

```
/release v1.4.0
# → github-release-engineer runs steps 1–11
# → on success, calls /homebrew-formula-updater
# → homebrew-formula-updater reads the artifact URLs and checksums
#   from the step report and updates the tap formula
```

Each skill owns exactly one domain. The release engineer does not know how to write a Homebrew formula. `homebrew-formula-updater` does not know how to watch a CI run. They compose by contract: the release engineer's step report provides the artifact URLs and checksums that downstream skills consume.

## Install

Symlink the skill into your skills directory so updates to the repo are picked up automatically:

```bash
mkdir -p ~/.claude/skills/release
ln -s "$(pwd)/prompts/release-skill.md" ~/.claude/skills/release/SKILL.md
```

## Usage

```
/release <version>                  # Full release for the given version tag
/release                            # Resolve version automatically, then run full release
/release --dry-run <version>        # Pre-flight and tag validation only; no push, no CI
/release --allow-ahead <version>    # Allow releasing when local branch is ahead of remote
```

### Options

| Flag | Default | Description |
|---|---|---|
| `--dry-run` | — | Validate version, changelog, and tag; stop before pushing |
| `--pre-release` | — | Mark the GitHub release as pre-release (`prerelease=true`, `draft=false`) |
| `--draft` | — | Create the release as a draft (`draft=true`, `prerelease=false`) |
| `--allow-ahead` | — | Allow releasing when local branch is ahead of remote |
| `--ci-timeout <minutes>` | `30` | CI watch timeout in minutes |
| `--ci-discovery-timeout <seconds>` | `60` | Seconds to wait for a CI run to appear after tag push |
| `--asset-timeout <minutes>` | `10` | Minutes to wait for release assets to be attached |
| `--poll-interval <seconds>` | `30` | Seconds between CI and asset status polls |
| `--retries <n>` | `2` | Max CI diagnose-and-retry cycles |

## Files

- [`prompts/release-skill.md`](prompts/release-skill.md): The `/release` skill — symlink to `~/.claude/skills/release/SKILL.md` or the equivalent path for your Agent Skills-compatible tool

## Design Philosophy

The GitHub release process has a natural boundary: everything that happens in the GitHub and GitHub Actions surface area. That boundary is where this skill starts and stops.

Distribution is not release. Publishing to Homebrew, apt, or a Docker registry is a separate concern with separate failure modes, separate credentials, and separate rollback procedures. Folding distribution into the release engineer would couple two unrelated responsibility domains. Instead, distribution is pluggable: each downstream target gets its own skill, its own retry logic, and its own rollback path.

This mirrors the [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) design principle: define the interface contract upfront, assign ownership cleanly, and compose at the boundary rather than merging concerns.

## License

[MIT](LICENSE)
