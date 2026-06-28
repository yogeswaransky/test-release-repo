# Automated Release Workflow

Automates main release and hotfix flows via GitHub Actions.

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

#### Concurrency

Runs are serialized per release key:

- `component-release-<release_type>-<release_version>`

This prevents two runs for the same version/type from racing on branches/tags.

#### Release Modes

**Main release — auto-complete**
1. Validates version format
2. `git flow release start` → changelog → `release publish` → `release finish`
3. Pushes to `main`, `develop`, and tags
4. Release is complete in a single run

**Main release — approvable**
1. `git flow release start` → changelog → `release publish`
2. Creates a PR: `release/<version>` → `develop`
3. Stops and waits for PR approval
4. On approval, the second workflow finishes the release

**Hotfix** (always auto-complete)
1. Requires `source_branch` input
2. Validates `source_branch` against `^[A-Za-z0-9._/-]+$`
3. Creates/uses `hotfix/<version>` from `source_branch`, updates changelog, merges back only into `source_branch`
4. Creates tag `<version>`
5. Pushes only `source_branch` and tag (main/develop are NOT pushed)

### 2. Component Release Finish On Approval (`component-release-finish-on-approval.yml`)

Triggered automatically when a `release/*` PR targeting `develop` is approved.

#### What it does
1. Checks that the PR review decision is `APPROVED`
2. Checks out the `release/*` branch from remote
3. Runs `git flow release finish` (merges to main + develop, creates tag)
4. Pushes `main`, `develop`, and tags
5. Closes the PR with a comment

## Failure Cleanup Safety

On failure, cleanup only removes refs created by the current run:

- Deletes tag only if the run created that tag
- Deletes `release/<version>` only if the run created that local release branch
- Deletes `hotfix/<version>` only if the run created that local hotfix branch
- Skips cleanup entirely if repository checkout is unavailable

## Prerequisites

### Repository Settings
- **Settings → Actions → General → Workflow permissions**: Select "Read and write permissions"
- **Settings → Actions → General**: Check "Allow GitHub Actions to create and approve pull requests"

### Branch Protection (for approvable mode)
- Enable "Require a pull request before merging" on `develop`
- Enable "Require approvals" with at least 1 required reviewer
- If workflow must push directly to `develop` (auto-complete flow), configure appropriate bypass for your automation actor

## Version Format

Accepted formats: `1.0.0`, `1.2.3-rc1`, `3.0.0v1`, `1.0.0.beta2`

Must start with `major.minor.patch` (digits), with an optional alphanumeric suffix.

## Flow Diagrams

### Auto-complete Release
```
Dispatch → validate version
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
Dispatch → validate version → validate source_branch (required + safe charset)
  → create/use hotfix branch from source_branch → changelog
  → merge hotfix into source_branch → tag
  → push source_branch + tag → ✅ Done
```
