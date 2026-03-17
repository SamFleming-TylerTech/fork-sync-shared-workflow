# Fork Action Sync Templates

Reusable GitHub Actions workflows for managing forked third-party GitHub Actions with automated upstream sync, tag monitoring, and security scanning.

## Overview

When you fork a third-party GitHub Action to pin it for supply chain security, you need to keep it in sync with upstream while reviewing changes before they reach your workflows. These reusable workflows automate that process.

Each fork contains thin caller workflows (~20 lines) that reference these reusable workflows via `workflow_call`. Centralized maintenance means bug fixes and detection improvements propagate to all forks automatically via the floating `@v1` tag.

Forks are bootstrapped by [`fork-action.sh`](https://github.com/SamFleming-TylerTech/fork-project-workflow/blob/main/fork-action.sh), which creates the caller workflows, branch structure, and manifest.

## Workflows

### sync-upstream.yml

Syncs the fork's `upstream-tracking` branch with the upstream repository's default branch.

**Inputs:**

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `upstream_owner` | string | yes | Owner of the upstream repository |
| `upstream_repo` | string | yes | Name of the upstream repository |
| `default_branch` | string | yes | Default branch of the upstream repository |
| `force` | boolean | no | Force sync even if already up to date (default: false) |

**What it does:**
- Checks out the fork's `upstream-tracking` branch
- Fetches the upstream default branch
- Attempts a fast-forward merge
- Opens an issue if divergence is detected (non-fast-forward)
- Creates a PR from `upstream-tracking` to the fork's default branch with diff stats and security review checklist

### sync-tags.yml

Monitors upstream tags for new releases, mutations (supply chain risk), and deletions.

**Inputs:**

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `upstream_owner` | string | yes | Owner of the upstream repository |
| `upstream_repo` | string | yes | Name of the upstream repository |

**What it does:**
- Compares upstream and fork tags using dereferenced commit SHAs (avoids false positives from annotated vs lightweight tag differences)
- Creates issues for new upstream releases with security review checklists
- Creates security alerts for tag mutations (upstream tag now points to different commit)
- Creates notices for tag deletions (upstream removed a tag)

### security-scan.yml

Runs security analysis on pull requests labeled `upstream-sync`.

**No inputs required** -- the pull request context passes through from the caller.

**What it does:**
- Dependency review (fails on high severity)
- CodeQL analysis with security-extended queries
- Diff summary with risk assessment posted as PR comment

## Usage

### Caller workflow examples

**sync-upstream.yml** (in your fork):
```yaml
name: Sync Upstream
on:
  schedule:
    - cron: '0 6 * * 1'
  workflow_dispatch:
    inputs:
      force:
        description: 'Force sync even if already up to date'
        required: false
        type: boolean
        default: false
concurrency:
  group: sync-upstream
  cancel-in-progress: false
jobs:
  sync:
    uses: SamFleming-TylerTech/fork-action-sync-templates/.github/workflows/sync-upstream.yml@v1
    with:
      upstream_owner: original-owner
      upstream_repo: original-repo
      default_branch: main
      force: ${{ inputs.force || false }}
    secrets: inherit
    permissions:
      contents: write
      pull-requests: write
      issues: write
```

**sync-tags.yml** (in your fork):
```yaml
name: Sync Tags
on:
  schedule:
    - cron: '0 6 * * 3'
  workflow_dispatch:
concurrency:
  group: sync-tags
  cancel-in-progress: false
jobs:
  check-tags:
    uses: SamFleming-TylerTech/fork-action-sync-templates/.github/workflows/sync-tags.yml@v1
    with:
      upstream_owner: original-owner
      upstream_repo: original-repo
    secrets: inherit
    permissions:
      contents: read
      issues: write
```

**security-scan.yml** (in your fork):
```yaml
name: Security Scan
on:
  workflow_run:
    workflows: ["Sync Upstream"]
    types: [completed]
concurrency:
  group: security-scan
  cancel-in-progress: false
jobs:
  scan:
    if: github.event.workflow_run.conclusion == 'success'
    uses: SamFleming-TylerTech/fork-action-sync-templates/.github/workflows/security-scan.yml@v1
    with:
      default_branch: main
    secrets: inherit
    permissions:
      contents: read
      pull-requests: write
      statuses: write
      checks: write
      security-events: write
```

## Versioning

This repo uses semantic versioning with a floating major version tag:

| Change Type | Example | Version Bump | Fork Action |
|---|---|---|---|
| Bug fix | Fix SHA comparison | `v1.x.y` patch | None -- auto-propagates via `@v1` |
| New optional input | Add `dry_run` | `v1.x.0` minor | None -- auto-propagates |
| Remove/rename input | Rename `force` | `v2.0.0` major | Update callers, re-run fork-action.sh |

### Releasing a non-breaking update

```bash
git tag -a v1.1.0 -m "Improved tag mutation detection"
git tag -f v1 -m "Latest v1.x release"
git push origin v1.1.0 && git push origin v1 --force
```

## Known Limitations

### CodeQL is limited to JavaScript/TypeScript

CodeQL analysis only runs on JavaScript and TypeScript codebases. Actions written in other languages (Go, Bash, Docker) still receive dependency review and diff summary checks, but no static analysis. CodeQL results are informational only and excluded from the aggregate security-scan status.

### Concurrency blocks must be in callers only

GitHub Actions rejects workflows when both the caller and the reusable workflow define `concurrency` blocks. All `concurrency` configuration belongs in the caller workflows, not in the reusable templates.

### Reusable workflow job names are prefixed

When called from a caller, job names are prefixed with the caller's job name. For example, if the caller job is named `scan` and the reusable workflow has a job `dependency-review`, the check name becomes `scan / dependency-review`. Branch protection rules must use the prefixed names.

## Requirements

- The templates repo must be **public** for cross-repo `workflow_call` to work
- Caller workflows must include `secrets: inherit` to pass `GITHUB_TOKEN`
- Caller workflows must specify appropriate `permissions` at the job level
