# Fork Action Sync Templates

Reusable GitHub Actions workflows for managing forked repositories with automated upstream sync, tag monitoring, and security scanning.

## Overview

These templates are used by forks managed via [fork-action.sh](https://github.com/tyler-technologies-oss/fork-project-workflow). Each fork contains thin caller workflows (~20 lines) that reference these reusable workflows, enabling centralized maintenance of sync logic.

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
  pull_request:
    types: [opened, synchronize, labeled]
concurrency:
  group: security-scan-${{ github.event.pull_request.number }}
  cancel-in-progress: true
jobs:
  scan:
    uses: SamFleming-TylerTech/fork-action-sync-templates/.github/workflows/security-scan.yml@v1
    secrets: inherit
    permissions:
      contents: read
      pull-requests: write
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

### Security Scan does not auto-trigger on workflow-created PRs

GitHub Actions prevents `GITHUB_TOKEN`-initiated events from triggering other workflows (to avoid infinite loops). Since Sync Upstream creates the PR using `GITHUB_TOKEN`, the `pull_request.opened` event does **not** fire for the Security Scan workflow.

**When the Security Scan DOES trigger:**

| Action | Token | Triggers Scan? |
|--------|-------|---------------|
| Sync Upstream creates PR | `GITHUB_TOKEN` | No |
| Human adds a label in GitHub UI | Human PAT | Yes (`labeled`) |
| Human pushes to `upstream-tracking` | Human PAT | Yes (`synchronize`) |
| Human closes/reopens PR | Human PAT | No (not in trigger list) |

**Practical impact:** The scan does not run until a human interacts with the PR. Since human review is required before merging upstream changes anyway, this is a minor gap -- the reviewer's first interaction with the PR (e.g., adding a label) triggers the scan.

**Possible future fix:** Add a `workflow_run` trigger to the Security Scan caller so it fires when Sync Upstream completes. This requires modifying the reusable workflow to resolve PR context from the GitHub API (since `workflow_run` events don't carry `pull_request` context).

### Concurrency blocks must be in callers only

GitHub Actions rejects workflows when both the caller and the reusable workflow define `concurrency` blocks. All `concurrency` configuration belongs in the caller workflows, not in the reusable templates.

### Reusable workflow job names are prefixed

When called from a caller, job names are prefixed with the caller's job name. For example, if the caller job is named `scan` and the reusable workflow has a job `dependency-review`, the check name becomes `scan / dependency-review`. Branch protection rules must use the prefixed names.

## Requirements

- The templates repo must be **public** for cross-repo `workflow_call` to work
- Caller workflows must include `secrets: inherit` to pass `GITHUB_TOKEN`
- Caller workflows must specify appropriate `permissions` at the job level
