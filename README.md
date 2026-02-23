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

### Security Scan auto-trigger requires a GitHub App (current)

The Sync Upstream workflow uses a GitHub App token (`actions/create-github-app-token`) to create PRs. PRs created by a GitHub App identity trigger `pull_request` events normally, so the Security Scan auto-fires without manual intervention.

**Required secrets (per-repo or org-level):**

| Secret | Description |
|--------|-------------|
| `FORK_SYNC_APP_ID` | GitHub App ID (numeric) |
| `FORK_SYNC_APP_PRIVATE_KEY` | GitHub App private key (PEM format) |

**GitHub App setup:**
1. Create a GitHub App (Settings > Developer settings > GitHub Apps)
2. Permissions: Contents (read/write), Pull requests (read/write), Issues (read/write)
3. Webhook: disabled (not needed)
4. Install the App on the target org or repos
5. Add the App ID and private key as org-level secrets (inherited by all forks)

**At scale:** One App installation per org. Two org-level secrets. Zero per-fork configuration.

#### Rollback: removing the GitHub App dependency

If the GitHub App cannot be maintained, revert to `GITHUB_TOKEN` by checking out the `v1.0.5` tag:

```
v1.0.5  -- last version using GITHUB_TOKEN (no App dependency)
v1.1.0  -- first version using GitHub App token
```

To roll back the floating `v1` tag:
```bash
cd fork-action-sync-templates
git tag -f v1 v1.0.5
git push origin v1 --force
```

All forks using `@v1` will immediately pick up the rollback. The `FORK_SYNC_APP_ID` and `FORK_SYNC_APP_PRIVATE_KEY` secrets can then be removed.

**Impact of rollback:** The Security Scan will no longer auto-trigger when Sync Upstream creates a PR. It will still trigger on any human interaction with the PR:

| Action | Triggers Scan? |
|--------|---------------|
| Sync Upstream creates PR (GITHUB_TOKEN) | No |
| Human adds a label in GitHub UI | Yes (`labeled`) |
| Human pushes to `upstream-tracking` | Yes (`synchronize`) |

For automated security monitoring at scale, this means scan results are not available until a human touches each PR. If this is acceptable, the rollback is safe. If unattended scanning is required, the GitHub App is necessary.

### Concurrency blocks must be in callers only

GitHub Actions rejects workflows when both the caller and the reusable workflow define `concurrency` blocks. All `concurrency` configuration belongs in the caller workflows, not in the reusable templates.

### Reusable workflow job names are prefixed

When called from a caller, job names are prefixed with the caller's job name. For example, if the caller job is named `scan` and the reusable workflow has a job `dependency-review`, the check name becomes `scan / dependency-review`. Branch protection rules must use the prefixed names.

## Requirements

- The templates repo must be **public** for cross-repo `workflow_call` to work
- Caller workflows must include `secrets: inherit` to pass `GITHUB_TOKEN`
- Caller workflows must specify appropriate `permissions` at the job level
