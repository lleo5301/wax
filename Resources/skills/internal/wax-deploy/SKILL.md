---
name: wax-deploy
description: Wax deployment orchestration skill. Use when releasing new versions of Wax across npm, Homebrew, OpenClaw, OpenCode, or Claude Code channels. Covers version bumping, congruency validation, binary building, staged publishing, and rollback procedures.
---

# Wax Deployment

## Overview

Wax deploys to 5 channels from a single source of truth. This skill ensures predictable, resilient, and congruent releases across all channels.

## Deployment Channels

| Channel | Artifact | Location |
|---------|----------|----------|
| npm | `waxmcp` package | `Resources/npm/waxmcp/` |
| Homebrew | `wax.rb` formula | `Resources/npm/waxmcp/homebrew-wax/` |
| OpenClaw | `@wax/openclaw-wax-memory` plugin | `Resources/openclaw/wax-memory-plugin/` |
| OpenCode | `.pi` extension + agents | `.pi/extensions/wax-agents/` |
| Claude Code | MCP server registration | `wax-cli mcp install` |

## Quick Reference

### One-command release

```bash
# Full pipeline: bump → build → validate → commit message
scripts/release-all.sh 0.1.23

# Then commit, tag, and publish:
git add -A && git commit -m "release: waxmcp v0.1.23"
git tag waxmcp-v0.1.23 && git push origin main --tags
scripts/publish-all.sh --tag next
```

### Check current state

```bash
scripts/validate-congruency.sh
# or JSON output for CI:
scripts/validate-congruency.sh --json
```

### Dry-run everything first

```bash
scripts/release-all.sh 0.1.23 --dry-run
scripts/publish-all.sh --tag next --dry-run
```

## Release Pipeline

### Phase 1: Bump

`scripts/bump-version.sh` atomically updates all 7 version surfaces:

1. `Resources/npm/waxmcp/package.json`
2. `Sources/WaxMCPServer/main.swift`
3. `Resources/openclaw/wax-memory-plugin/package.json` (version)
4. `Resources/openclaw/wax-memory-plugin/package.json` (waxmcp dependency)
5. `Resources/npm/waxmcp/homebrew-wax/Formula/wax.rb` (URL)
6. `Resources/npm/waxmcp/homebrew-wax/Formula/wax.rb` (SHA256)
7. `.pi/extensions/wax-agents/package.json`

```bash
scripts/bump-version.sh 0.1.23        # exact version
scripts/bump-version.sh patch         # or minor/major
scripts/bump-version.sh 0.1.23 --dry-run
```

### Phase 2: Build

`scripts/release-all.sh` builds darwin-arm64 and darwin-x64 binaries:

```bash
scripts/release-all.sh 0.1.23
# Skip build if binaries already exist:
scripts/release-all.sh 0.1.23 --skip-build
```

### Phase 3: Validate

`scripts/validate-congruency.sh` asserts all surfaces match:

```bash
scripts/validate-congruency.sh              # human-readable
scripts/validate-congruency.sh --json       # machine-readable
scripts/validate-congruency.sh --quiet      # CI-friendly (exit 0/1 only)
```

### Phase 4: Publish (staged)

`scripts/publish-all.sh` supports canary → soak → promote workflow:

```bash
# Publish to canary
scripts/publish-all.sh --tag next

# After soak period, promote to latest
scripts/publish-all.sh --promote

# Preview without publishing
scripts/publish-all.sh --tag next --dry-run
```

## Version Congruency

All channels must share the same version. The canonical source is `Resources/npm/waxmcp/package.json`.

### Common incongruency scenarios

| Symptom | Cause | Fix |
|---------|-------|-----|
| openclaw ≠ npm | Plugin not bumped after waxmcp release | `scripts/bump-version.sh $(npm version)` |
| homebrew ≠ npm | Formula manually edited, not via script | `scripts/bump-version.sh` recomputes URL + SHA |
| opencode ≠ npm | Extension version never tracked | `scripts/bump-version.sh` aligns it |

### CI integration

Add to `.github/workflows/release-waxmcp.yml` before publish:

```yaml
- name: Validate version congruency
  run: scripts/validate-congruency.sh --quiet
```

## Rollback

### Before publish (local)

```bash
git checkout -- Resources/npm/waxmcp/package.json \
  Sources/WaxMCPServer/main.swift \
  Resources/openclaw/wax-memory-plugin/package.json \
  .pi/extensions/wax-agents/package.json \
  Resources/npm/waxmcp/homebrew-wax/Formula/wax.rb
```

### After npm publish (next tag)

```bash
# Unpublish within 24 hours
npm unpublish waxmcp@0.1.23
npm unpublish @wax/openclaw-wax-memory@0.1.23
```

### After promote to latest

```bash
# Republish previous version as latest
npm dist-tag add waxmcp@0.1.22 latest
npm dist-tag add @wax/openclaw-wax-memory@0.1.22 latest
```

## Troubleshooting

### "Git working tree is dirty"

Commit or stash changes before running `release-all.sh`:
```bash
git stash
scripts/release-all.sh 0.1.23
git stash pop
```

Or use `--allow-dirty` (not recommended for production releases).

### "Version incongruency detected"

Run the bump script to align all surfaces:
```bash
scripts/bump-version.sh $(node -p "require('./Resources/npm/waxmcp/package.json').version")
```

### "npm publish fails with E403"

Version already published. Check:
```bash
npm view waxmcp versions --json | tail -5
```

### Homebrew SHA mismatch

The SHA in the formula is computed from local `git archive`. After pushing the tag, verify:
```bash
curl -sL https://github.com/christopherkarani/Wax/archive/refs/tags/waxmcp-v0.1.23.tar.gz | shasum -a 256
```

If different, update the formula manually or re-run `bump-version.sh` after the tag is pushed.

## Design Decisions

- **Single source of truth**: npm package version is canonical; all others derive from it
- **Atomic bumps**: `bump-version.sh` updates all surfaces in one invocation
- **Dry-run everywhere**: Every script supports `--dry-run` for safe preview
- **Staged publishing**: `next` tag for canary, `latest` for production
- **Fail-fast**: `set -euo pipefail` on all scripts; validation gates before publish
- **Delegate, don't duplicate**: New scripts call existing `Resources/scripts/` helpers
