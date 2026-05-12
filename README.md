# Automated Release Workflow

Automates the git-flow release and hotfix process via GitHub Actions.

## Workflows

### 1. Component Release (`component-release.yml`)

Triggered manually from the **Actions** tab via `workflow_dispatch`.

#### Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `release_version` | Yes | Release version (e.g. `1.0.0`, `3.0.0v1`) |
| `release_type` | Yes | `main` or `hotfix` |
| `release_mode` | Yes | `approvable` or `auto-complete` (ignored for hotfix) |
| `source_branch` | Hotfix only | Support branch for hotfix (e.g. `support/1.x`) |

#### Release Modes

**Main release — auto-complete** (maintainers only)
1. Validates actor has maintainer/owner permission
2. `git flow release start` → changelog → `release publish` → `release finish`
3. Pushes to `main`, `develop`, and tags
4. Release is complete in a single run

**Main release — approvable** (anyone)
1. `git flow release start` → changelog → `release publish`
2. Creates a PR: `release/<version>` → `develop`
3. Stops and waits for PR approval
4. On approval, the second workflow finishes the release

**Hotfix** (always auto-complete)
1. Requires `source_branch` input (support branch)
2. `git flow hotfix start` → changelog → `hotfix finish`
3. Pushes only the source branch and tag (main/develop are NOT pushed)

### 2. Component Release Finish On Approval (`component-release-finish-on-approval.yml`)

Triggered automatically when a `release/*` PR targeting `develop` is approved.

#### What it does
1. Checks that the PR review decision is `APPROVED`
2. Checks out the `release/*` branch from remote
3. Runs `git flow release finish` (merges to main + develop, creates tag)
4. Pushes `main`, `develop`, and tags
5. Closes the PR with a comment

## Prerequisites

### Repository Settings
- **Settings → Actions → General → Workflow permissions**: Select "Read and write permissions"
- **Settings → Actions → General**: Check "Allow GitHub Actions to create and approve pull requests"

### Branch Protection (for approvable mode)
- Enable "Require a pull request before merging" on `develop`
- Enable "Require approvals" with at least 1 required reviewer

## Version Format

Accepted formats: `1.0.0`, `1.2.3-rc1`, `3.0.0v1`, `1.0.0.beta2`

Must start with `major.minor.patch` (digits), with an optional alphanumeric suffix.

## Flow Diagrams

### Auto-complete Release
```
Dispatch → validate version → authorize (maintainer check)
  → release start → changelog → release publish → release finish
  → push main + tags + develop → ✅ Done
```

### Approvable Release
```
Dispatch → validate version
  → release start → changelog → release publish
  → create PR (release → develop) → ⏸️ Wait for approval

PR approved → finish-on-approval workflow triggers
  → release finish → push main + tags + develop
  → close PR → ✅ Done
```

### Hotfix
```
Dispatch → validate version → validate source_branch
  → checkout main + source branch → hotfix start → changelog
  → hotfix finish → push source branch + tag → ✅ Done
```
