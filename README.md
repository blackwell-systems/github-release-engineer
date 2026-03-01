# github-release-engineer

[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)

A Claude Code skill that automates the full GitHub release process end-to-end — from tagging through CI, artifact production, and release verification.

## What It Is

`github-release-engineer` is a `/release` skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). It handles everything on the GitHub side of a release: creating and pushing tags, monitoring CI, diagnosing and fixing failures, waiting for goreleaser artifacts, and verifying the final release state.

It lives in the blackwell-systems ecosystem alongside [scout-and-wave](https://github.com/blackwell-systems/scout-and-wave) and [agentic-cold-start-audit](https://github.com/blackwell-systems/agentic-cold-start-audit).

## What It Does Not Do

`github-release-engineer` is scoped to the GitHub side of the release lifecycle. It does not update distribution targets — Homebrew formulae, apt repositories, Docker Hub manifests, or any other downstream package registry. That work is explicitly out of scope.

Distribution updates are handled by companion skills. The release engineer calls them as a step. They own their domain; the release engineer owns GitHub.

## Process

The skill follows a fixed sequence. Each step gates the next:

1. **Pre-flight** — Verify the working tree is clean, the branch is up to date with the remote, and there are no uncommitted changes. Abort with a clear error if any check fails.
2. **Version resolution** — Determine the release version: read from argument, `VERSION` file, `Cargo.toml`, `package.json`, or prompt the user. Validate it is a valid semver string.
3. **Changelog verification** — Confirm an entry exists for the version in `CHANGELOG.md` (or equivalent). Warn but do not abort if no changelog is found; the user decides.
4. **Tag** — Create a signed, annotated git tag (`git tag -s vX.Y.Z -m "Release vX.Y.Z"`). Verify the tag before pushing.
5. **Push tag** — Push the tag to the remote. Confirm the push succeeded.
6. **Watch CI** — Poll the GitHub Actions workflow run triggered by the tag. Stream status until the run reaches a terminal state (success, failure, cancelled). Timeout after a configurable maximum (default 30 minutes).
7. **Diagnose and fix failures** — If the run fails, fetch the failed job logs, identify the root cause, propose a fix, apply it (with user confirmation), amend or add a commit, delete and re-push the tag, and re-enter the CI watch loop. Repeat up to a configurable retry limit.
8. **Wait for goreleaser artifacts** — After CI passes, poll the GitHub release for goreleaser-produced assets. Wait until all expected artifacts are attached or the wait times out.
9. **Verify the release** — Confirm the release is published (not draft, not pre-release unless explicitly flagged), the tag matches, assets are present and non-zero in size, and the release notes are populated.
10. **Report** — Emit a structured summary: version, tag, CI run URL, artifact list with sizes, release URL, and any warnings encountered during the run.

## Composition with Companion Skills

The release engineer handles the GitHub side only. When the GitHub release is verified and the step report is emitted, the caller can invoke companion skills for distribution:

```
/release v1.4.0
# → github-release-engineer runs steps 1–10
# → on success, calls /homebrew-formula-updater
# → homebrew-formula-updater fetches the goreleaser-produced darwin artifact,
#   computes the sha256, opens a PR against the tap repo
```

Each skill owns exactly one domain. The release engineer does not know how to write a Homebrew formula. `homebrew-formula-updater` does not know how to watch a CI run. They compose by contract: the release engineer's step report provides the artifact URLs and checksums that downstream skills consume.

## Install

Copy the skill to your global Claude Code commands directory:

```bash
cp prompts/release-skill.md ~/.claude/commands/release.md
```

## Usage

```
/release <version>           # Full release for the given version tag
/release                     # Resolve version automatically, then run full release
/release --dry-run <version> # Pre-flight and tag validation only; no push, no CI
```

### Options

| Flag | Description |
|---|---|
| `--dry-run` | Validate version, changelog, and tag; stop before pushing |
| `--no-sign` | Skip GPG signing (not recommended; requires explicit acknowledgment) |
| `--pre-release` | Mark the GitHub release as pre-release |
| `--timeout <minutes>` | CI watch timeout in minutes (default: 30) |
| `--retries <n>` | Max CI fix-and-retry cycles (default: 2) |

## Files

- [`prompts/release-skill.md`](prompts/release-skill.md): The `/release` Claude Code skill (copy to `~/.claude/commands/release.md`)

## Design Philosophy

The GitHub release process has a natural boundary: everything that happens in the GitHub and GitHub Actions surface area. That boundary is where this skill starts and stops.

Distribution is not release. Publishing to Homebrew, apt, or a Docker registry is a separate concern with separate failure modes, separate credentials, and separate rollback procedures. Folding distribution into the release engineer would couple two unrelated responsibility domains. Instead, distribution is pluggable: each downstream target gets its own skill, its own retry logic, and its own rollback path.

This mirrors the scout-and-wave design principle: define the interface contract upfront, assign ownership cleanly, and compose at the boundary rather than merging concerns.

## License

[MIT](LICENSE)
