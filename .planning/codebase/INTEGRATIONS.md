# External Integrations

**Analysis Date:** 2026-03-17

## APIs & External Services

**GitHub API:**

- GitHub REST API - Primary integration for workflow run queries and commit verification
  - SDK/Client: `@actions/github` 6.0.1 (wraps Octokit)
  - Auth: GitHub token passed via `gh-token` input (action.yml line 5)
  - Token set to environment: `process.env.GITHUB_TOKEN` (nx-set-shas.ts line 12)
  - Used by: `github.getOctokit(process.env.GITHUB_TOKEN)` (nx-set-shas.ts line 185)

**Endpoints Used:**

- `GET /repos/{owner}/{repo}/actions/runs/{run_id}` - Fetch current workflow run to determine workflow_id if not provided (nx-set-shas.ts line 188)
- `GET /repos/{owner}/{repo}/actions/workflows/{workflow_id}/runs` - Query workflow runs by branch, event type, and status (nx-set-shas.ts line 202)
- `GET /repos/{owner}/{repo}/commits/{commit_sha}` - Verify commit exists on GitHub (nx-set-shas.ts line 255)
- `GET /repos/{owner}/{repo}/commits` - List commits on a branch to verify a specific commit exists in branch history (nx-set-shas.ts line 262)

**Query Parameters:**

- `branch` - Target branch name
- `workflow_id` - GitHub Actions workflow identifier
- `event` - Workflow trigger event type (push, pull_request, workflow_dispatch, etc.)
- `status` - Workflow run status (success, failure, etc.)
- `per_page` - Pagination limit (set to 100 for commit listing, nx-set-shas.ts line 266)

## Data Sources

**Git Commands:**

- Uses local git commands for fast SHA resolution:
  - `git rev-parse HEAD` - Get current HEAD SHA (nx-set-shas.ts line 40)
  - `git merge-base <remote>/<branch> HEAD` - Find common ancestor for PR base SHA (nx-set-shas.ts lines 54-59)
  - `git rev-parse HEAD^1` - Get parent commit for merge_group events (nx-set-shas.ts line 63)
  - `git hash-object -t tree /dev/null` - Get empty git tree hash as fallback (nx-set-shas.ts lines 104-107)
  - `git cat-file -e <commit>` - Verify commit exists locally (nx-set-shas.ts line 250)

**Repository Information:**

- Source: GitHub Actions `github.context` (nx-set-shas.ts lines 8-11)
- Contains: repo name, owner, event name, run ID
- Used for API calls and git operations

## Authentication & Identity

**Auth Provider:**

- GitHub Token (standard GitHub Actions token or custom provided)
- Implementation: Token-based authentication passed to Octokit
- Scope: Token must have permissions to read workflow runs and commits
  - GitHub documentation (from README.md line 14): "Permissions in v2+"

**Event Context:**

- GitHub Actions context provides:
  - Pull request metadata (base branch, merged status)
  - Merge group information
  - Event name (push, pull_request, merge_group, workflow_dispatch, etc.)

## Webhooks & Callbacks

**Incoming:**

- None - This is an action that reads data from GitHub, doesn't receive webhooks

**Outgoing:**

- None - Action only queries existing workflow runs and commits, doesn't trigger events

## Environment Configuration

**Required env vars:**

- `GITHUB_TOKEN` - GitHub authentication token (set from `gh-token` input)
  - Default: `${{ github.token }}` (action.yml line 7)
  - Custom token can be provided via `gh-token` input

**Optional env vars:**

- `FORCE_COLOR` - Set to 'true' in pre-commit hook for colored output (tools/pre-commit.ts lines 15, 21, 36)

**Action Inputs (Environment):**

- `gh-token` - GitHub token (default: github.token)
- `main-branch-name` - Target main branch (default: 'main')
- `remote` - Git remote name (default: 'origin')
- `set-environment-variables-for-job` - Export NX_BASE and NX_HEAD (default: 'true')
- `error-on-no-successful-workflow` - Hard error vs warning (default: 'false')
- `fallback-sha` - SHA to use if no successful workflow found (optional)
- `last-successful-event` - Event type to track (default: 'push')
- `working-directory` - Repo location (default: '.')
- `workflow-id` - Workflow identifier (optional, derived if not provided)
- `use-previous-merge-group-commit` - Use parent commit for merge_group (default: 'true')

All inputs passed via GitHub Actions YAML configuration (action.yml lines 4-34).

**Action Outputs:**

- `base` - Base SHA for nx affected (nx-set-shas.ts line 160)
- `head` - Head SHA for nx affected (nx-set-shas.ts line 161)
- `noPreviousBuild` - Set to 'true' if no previous successful build found (nx-set-shas.ts line 128)

## Workflow Integration Points

**Trigger Detection:**

- `github.context.eventName` - Detects workflow trigger type:
  - `pull_request` / `pull_request_target` - PR workflow (nx-set-shas.ts line 46)
  - `merge_group` - Merge queue workflow (nx-set-shas.ts line 62)
  - Other events (push, workflow_dispatch) - Query workflow history

**PR Context:**

- Reads from `github.context.payload.pull_request`:
  - `base.ref` - Target branch name (nx-set-shas.ts line 56)
  - `merged` - Whether PR is merged (nx-set-shas.ts line 47)

**GitHub Actions Context:**

- `github.context.runId` - Current workflow run ID (nx-set-shas.ts line 8)
- `github.context.repo.owner` - Repository owner (nx-set-shas.ts line 9)
- `github.context.repo.repo` - Repository name (nx-set-shas.ts line 9)

## Failure Handling

**API Failures:**

- GitHub API requests wrapped in try-catch (nx-set-shas.ts lines 249-274)
- Failed commit verification returns false, continues to next commit
- If all commits fail verification, undefined is returned

**Workflow Run Resolution:**

- If no successful workflow run found:
  - Default behavior: Log warning, use HEAD~1 or empty tree hash
  - With `error-on-no-successful-workflow: true`: Hard error via `core.setFailed()`
  - With `fallback-sha` provided: Use fallback value

**Git Command Failures:**

- Git commands use `spawnSync` with error checking (nx-set-shas.ts line 40, etc.)
- No stderr output captured in most cases
- Status code checked for merge-base and rev-parse operations

---

_Integration audit: 2026-03-17_
