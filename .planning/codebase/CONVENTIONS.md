# Coding Conventions

**Analysis Date:** 2026-03-17

## Naming Patterns

**Files:**

- TypeScript source files use kebab-case for utilities, camelCase for core logic: `nx-set-shas.ts`, `pre-commit.ts`
- Workflow files use kebab-case: `test-integration.yml`, `format.yml`

**Functions:**

- Named functions use camelCase: `findSuccessfulCommit()`, `findExistingCommit()`, `commitExists()`, `stripNewLineEndings()`
- Async functions are clearly marked with `async` keyword
- Private/internal functions (helper functions within the module) follow same camelCase convention
- Utility functions grouped at module level follow descriptive names indicating their purpose

**Variables:**

- Constants declared at module scope use UPPER_SNAKE_CASE: `BASE_SHA`, `HEAD_SHA`, `GITHUB_TOKEN`
- Local variables use camelCase: `headResult`, `baseResult`, `commitSha`, `branchName`
- Configuration/input variables from GitHub Actions inputs use camelCase: `mainBranchName`, `errorOnNoSuccessfulWorkflow`, `lastSuccessfulEvent`, `workingDirectory`

**Types:**

- TypeScript types imported from `@actions/github/lib/utils` are used for strict typing: `InstanceType<typeof GitHub>`
- Avoid untyped parameters; use explicit types from dependencies
- Return types for async functions explicitly specify `Promise<Type>` or `Promise<Type | undefined>`

## Code Style

**Formatting:**

- Prettier 3.6.2 enforces formatting
- Configuration in `.prettierrc`:
  - `singleQuote: true` - Use single quotes for strings
  - `endOfLine: 'lf'` - Unix line endings
  - `tabWidth: 2` - 2-space indentation

**Linting:**

- No ESLint configuration present; Prettier handles formatting exclusively
- Pre-commit hook validates Prettier formatting before commit

**Spacing and Structure:**

- Blank lines separate logical sections within functions
- Multi-line operations (like `spawnSync` calls with options) are properly indented
- String templates used for dynamic values in messages and commands

## Import Organization

**Order:**

1. Node.js built-in modules (with `node:` prefix): `import * as core from '@actions/core'`, `import { spawnSync } from 'node:child_process'`
2. Third-party GitHub Actions packages: `import * as github from '@actions/github'`, `import { GitHub } from '@actions/github/lib/utils'`
3. Local utility functions follow as needed

**Path Aliases:**

- No path aliases configured in `tsconfig.json`
- Direct relative imports used within single files

**Style:**

- Use namespace imports for modules with multiple exports: `import * as core from '@actions/core'`
- Use destructuring for specific exports: `import { spawnSync } from 'node:child_process'`

## Error Handling

**Patterns:**

- Try-catch blocks wrap async API calls and command execution that may fail: `await octokit.request()`, `spawnSync()` with git commands
- Catch blocks check for existence of errors without type assertion: `catch { return false }` or `catch (e) { core.setFailed(e.message) }`
- GitHub Actions `core.setFailed()` is used to report errors and stop execution flow
- Synchronous failures in `spawnSync` checked via `status` field: `if (baseRes.status !== 0 || !baseRes.stdout)`

**Error Context:**

- Errors include contextual information in messages, e.g., branch name and workflow name
- Fallback behavior implemented when API calls fail (using empty tree hash or HEAD~1)
- User-facing warnings written to `process.stdout.write()` with full context

## Logging

**Framework:** `process.stdout.write()` - No logging framework used

**Patterns:**

- All user-facing messages written to stdout using `process.stdout.write()`
- Newlines explicitly written as separate calls or within template literals: `process.stdout.write('\n')`
- Multi-line messages constructed with multiple write calls for clarity
- Status messages include context: commit SHAs, warnings about missing workflows, etc.
- Output includes markers for warnings and notes: `WARNING:`, `NOTE:`, `HEAD~1 does not exist.`

**Environment Variables:**

- Logged when set for user reference: `core.exportVariable('NX_BASE', BASE_SHA)`
- Status messages confirm environment variables have been set

## Comments

**When to Comment:**

- Complex logic with non-obvious intent: "Both pull_request and pull_request_target events have the same payload structure"
- Links to issues for context: Comments reference GitHub issue numbers (#186)
- Explanations of edge cases: "Check if HEAD~1 exists, and if not, set BASE_SHA to the empty tree hash"
- Constants with domain-specific meaning: "4b825dc642cb6eb9a060e54bf8d69288fbee4904 is the expected result of hashing the empty tree"
- Conditional branches explaining why branches exist: "on some workflow runs we do not have branch property"

**JSDoc/TSDoc:**

- JSDoc comments precede functions with brief descriptions:
  ```typescript
  /**
   * Find last successful workflow run on the repo
   */
  async function findSuccessfulCommit(...): Promise<string | undefined>
  ```
- No parameter or return type documentation in JSDoc (types are explicit in function signature)

## Function Design

**Size:** Functions are focused and single-purpose:

- `findSuccessfulCommit()` queries GitHub API for workflow runs
- `findExistingCommit()` iterates through SHAs checking existence
- `commitExists()` performs a single validation check
- Helper functions like `stripNewLineEndings()` are 1-2 lines

**Parameters:**

- Functions receive all necessary parameters explicitly; no global state dependencies
- Multiple parameters grouped logically: owner, repo, branch, commitSha passed together to GitHub API calls
- Type hints used for all parameters: `branch: string`, `shas: string[]`, `octokit: InstanceType<typeof GitHub>`

**Return Values:**

- Functions return values needed by caller: SHAs, booleans for existence checks
- Async functions return `Promise<T | undefined>` to signal success/failure without throwing
- Void functions used for side effects only: `reportFailure()`, `core.setFailed()`

## Module Design

**Exports:**

- Single IIFE (Immediately Invoked Function Expression) wraps entire script: `(async () => { ... })()`
- No named exports; script executes directly on import
- Helper functions defined at module scope below main logic

**Barrel Files:**

- Not applicable; single-file module structure

**Top-level Execution:**

- Module-level code executes immediately: input variables read from `core.getInput()`, async function invoked
- GitHub Actions core and github modules used directly at module scope
- All side effects (logging, setting outputs, exporting variables) handled at top level

---

_Convention analysis: 2026-03-17_
