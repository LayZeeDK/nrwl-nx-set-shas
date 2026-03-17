# Architecture

**Analysis Date:** 2026-03-17

## Pattern Overview

**Overall:** GitHub Actions application with a single-entry-point design

**Key Characteristics:**

- Monolithic TypeScript entry point compiled to executable JavaScript
- Async/await driven with imperative control flow
- Direct shell command invocation via `spawnSync` for Git operations
- GitHub API integration via Octokit client
- Configuration via GitHub Actions input parameters
- No module separation or layering - all logic in one file

## Layers

**Input Layer:**

- Purpose: Extract and parse GitHub Actions inputs and context
- Location: `nx-set-shas.ts` (lines 7-24)
- Contains: Parameter extraction from `@actions/core` and `@actions/github`
- Depends on: `@actions/core`, `@actions/github`
- Used by: Main execution logic

**Git Operations Layer:**

- Purpose: Execute Git commands and parse results
- Location: `nx-set-shas.ts` (lines 40-61, 63-66, 98-119)
- Contains: `spawnSync` calls for `git rev-parse`, `git merge-base`, `git cat-file`, `git hash-object`
- Depends on: Node.js `child_process` module
- Used by: SHA resolution logic

**GitHub API Layer:**

- Purpose: Fetch workflow metadata and commit information from GitHub
- Location: `nx-set-shas.ts` (lines 177-275)
- Contains: `findSuccessfulCommit`, `findExistingCommit`, `commitExists` functions
- Depends on: Octokit client (`@actions/github`)
- Used by: SHA discovery logic

**Decision/Resolution Layer:**

- Purpose: Determine which SHA to use based on event context
- Location: `nx-set-shas.ts` (lines 28-137)
- Contains: Event type branching (pull_request, merge_group, default)
- Depends on: Input layer, Git operations layer, GitHub API layer
- Used by: Output layer

**Output Layer:**

- Purpose: Set GitHub Actions outputs and environment variables
- Location: `nx-set-shas.ts` (lines 139-161)
- Contains: `core.setOutput`, `core.exportVariable` calls
- Depends on: `@actions/core`
- Used by: GitHub workflow context

## Data Flow

**Pull Request / Pull Request Target Event:**

1. Extract event context and inputs from GitHub Actions
2. Parse HEAD SHA using `git rev-parse HEAD`
3. Fetch pull request base branch from event payload
4. Calculate merge-base between remote base branch and HEAD
5. Output merge-base SHA as BASE, HEAD SHA as HEAD

**Merge Group Event (with `use-previous-merge-group-commit` enabled):**

1. Extract event context and inputs
2. Parse HEAD SHA
3. Get previous commit in group using `git rev-parse HEAD^1`
4. Output previous commit SHA as BASE, HEAD SHA as HEAD

**Default Path (Push / Workflow Dispatch):**

1. Extract event context and inputs
2. Parse HEAD SHA
3. Call `findSuccessfulCommit` to fetch workflow metadata:
   - Query GitHub API for workflow runs on main branch with success status
   - Collect HEAD SHAs from all successful runs
4. Call `findExistingCommit` to find first SHA that still exists:
   - For each SHA in list: check if it exists in local repo and on remote branch
   - Return first SHA that passes both checks
5. If no successful commit found:
   - If `error-on-no-successful-workflow` true: report failure and exit
   - Otherwise: attempt fallback (fallback-sha input, HEAD~1, or empty tree hash)
6. Output determined BASE SHA and HEAD SHA

**State Management:**

- BASE_SHA and HEAD_SHA are module-level variables set during async execution
- Configuration comes entirely from GitHub Actions inputs (no state persistence)
- Each Action invocation is stateless relative to previous runs

## Key Abstractions

**SHA Resolution:**

- Purpose: Encapsulate logic to find a valid previous commit SHA
- Examples: `findSuccessfulCommit`, `findExistingCommit`, `commitExists`
- Pattern: Sequential search through candidates, returning first valid result

**Event Context Extraction:**

- Purpose: Normalize GitHub event payload access
- Examples: `github.context.eventName`, `github.context.payload.pull_request.base.ref`
- Pattern: Direct property access with null-coalescing fallbacks

**Command Execution:**

- Purpose: Execute Git commands and capture output
- Examples: `spawnSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf-8' })`
- Pattern: Use Node.js child_process with synchronous execution, UTF-8 encoding, exit code checking

## Entry Points

**GitHub Actions Execution:**

- Location: `nx-set-shas.ts` (lines 28-162)
- Triggers: Automatically when action is invoked in workflow (via `action.yml` entry point `dist/nx-set-shas.js`)
- Responsibilities:
  - Parse all input parameters
  - Determine correct SHA resolution path based on event type
  - Call appropriate Git commands and GitHub API endpoints
  - Handle error conditions (missing commits, no successful workflows)
  - Set output values and environment variables

**Pre-commit Hook:**

- Location: `tools/pre-commit.ts`
- Triggers: Git pre-commit hook (via Husky)
- Responsibilities:
  - Ensure build artifacts are up to date
  - Format code with Prettier
  - Stage any updated build files

## Error Handling

**Strategy:** Try-catch with fallback escalation

**Patterns:**

- `findSuccessfulCommit`: Throws error if no workflow found and `error-on-no-successful-workflow` is true
- `findExistingCommit`: Gracefully returns undefined if no valid commit found in list
- `commitExists`: Returns false on any exception (invalid SHA, network error, etc.)
- Fallback chain: User-provided SHA → HEAD~1 → Empty tree hash (with warnings logged to stdout)

## Cross-Cutting Concerns

**Logging:** Direct output to `process.stdout.write()` for step context, using `process.stdout.write` for newline control

**Validation:**

- Git command exit codes checked via `spawnSync` status property
- Commit existence validated via `git cat-file -e` and GitHub API requests
- Input parsing handled by `@actions/core` (getInput, getBooleanInput)

**Authentication:** GitHub token passed via environment variable `GITHUB_TOKEN` set from Action input

---

_Architecture analysis: 2026-03-17_
