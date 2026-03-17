# Testing Patterns

**Analysis Date:** 2026-03-17

## Test Framework

**Runner:**

- No unit test framework configured (Jest, Vitest, etc. not present)
- Testing performed via GitHub Actions workflows for integration testing
- Primary test workflow: `.github/workflows/test.yml`
- Integration test workflow: `.github/workflows/test-integration.yml`

**Test Execution:**

- Tests run on multiple platforms: `ubuntu-latest`, `macos-latest`, `windows-latest`
- Each test job compiles the code and executes the action in real GitHub Actions environment
- Fail-fast disabled to ensure all platforms are tested even if one fails

**Build Command:**

```bash
bun run build  # Compiles TypeScript → dist/nx-set-shas.js
```

## Test File Organization

**Location:**

- No dedicated test directory; integration tests embedded in workflow files
- Verification steps written inline in workflow step definitions
- Test logic uses shell bash commands and Node.js inline scripts

**Naming:**

- Workflow test files: `test.yml`, `test-integration.yml`
- Tests identified by `name` field in workflow steps

**Structure:**

```yaml
# Pattern from .github/workflows/test.yml
- name: Checkout [Pull Request]
  if: ${{ github.event_name == 'pull_request' }}
  ...

- name: Install
  run: bun install

- name: Compile
  run: bun run build

- name: Test default PR Workflow
  uses: ./        # Execute the action itself
  with:
    main-branch-name: ${{ github.base_ref }}

- name: Verify default PR Workflow
  shell: bash
  run: |
    BASE_SHA=$(echo $(git merge-base origin/${{github.base_ref}} HEAD))
    ...
```

## Test Structure

**Workflow Test Pattern:**

1. **Checkout:** Fetch code on appropriate event (PR vs Push)
2. **Setup:** Install Bun package manager
3. **Build:** Compile TypeScript to JavaScript
4. **Execute:** Run the GitHub Action itself (`uses: ./`)
5. **Verify:** Assert output environment variables match expected values

Example verification from `test.yml`:

```bash
# Verify environment variables set by the action
BASE_SHA=$(echo $(git merge-base origin/${{github.base_ref}} HEAD))
HEAD_SHA=$(git rev-parse HEAD)
node -e "if(process.env.NX_BASE == '${BASE_SHA}') console.log('Base set correctly'); else { throw new Error('Base not set correctly!');}"
node -e "if(process.env.NX_HEAD == '${HEAD_SHA}') console.log('Head set correctly'); else { throw new Error('Head not set correctly!');}"
```

**Assertion Pattern:**

- Assertions use Node.js command-line execution with inline scripts
- Conditional checks with `throw new Error()` for failures
- Simple `console.log()` for success messages

## Test Scenarios

**PR Event Tests:**

- File: `.github/workflows/test.yml` (lines 47-61)
- Trigger: `github.event_name == 'pull_request'`
- Validates: `NX_BASE` set to merge-base of PR branch with base branch
- Validates: `NX_HEAD` set to HEAD of PR branch

**Push Event Tests:**

- File: `.github/workflows/test.yml` (lines 63-81)
- Trigger: `github.event_name != 'pull_request'`
- Validates: `NX_BASE` set appropriately or empty string if not ancestor
- Validates: `NX_HEAD` set to current commit

**Integration Tests:**

- File: `.github/workflows/test-integration.yml`
- Workflow: Calls `.github/workflows/integration-test-workflow.yml`
- Working directory: `integration-test/`
- Tests action behavior in real workflow scenario

## Mocking

**Framework:** No mocking framework present

**Approach:**

- No unit test mocks; integration tests use real git operations
- Real GitHub API calls via `@actions/github` Octokit client
- Git commands executed directly via `spawnSync('git', [...])`
- Test uses real GitHub repository state

**Environment Setup:**

- GitHub Actions provides real context: `github.context.payload`, `process.env.GITHUB_TOKEN`
- No mock data factories or fixtures
- Tests execute in live GitHub Actions environment with real repository

## Test Execution Environment

**CI/CD Platform:** GitHub Actions

**Matrix Strategy:**

```yaml
strategy:
  matrix:
    runs-on: [ubuntu-latest, macos-latest, windows-latest]
  fail-fast: false # All platforms tested even if one fails
```

**Concurrency:**

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.number || github.ref }}
  cancel-in-progress: true # Cancel previous runs on same ref
```

## Code Coverage

**Requirements:** No coverage requirements enforced

**Measurement:** No coverage collection configured

**Tool:** Not applicable (integration testing approach)

## Pre-commit Testing

**Hook Location:** `.husky/pre-commit`

**Validation Steps:**

```bash
# 1. Build the action
npm run build

# 2. Stage any modified dist files
git add dist/

# 3. Format code
npm run format
```

**Execution:**

- File: `tools/pre-commit.ts`
- Framework: Node.js script with `execSync`
- UI: Uses `yoctocolors` for colored terminal output (with fallback for Windows compatibility)

**Failure Handling:**

```typescript
function printErrorAndExit(error) {
  console.log(error);
  console.log(`\n${red(bold(' Commit failed '))}\n`);
  process.exit(1);
}
```

## Build Verification

**Build Process:**

```bash
bun build ./nx-set-shas.ts --outdir ./dist --target node
```

**Source:** `nx-set-shas.ts` (TypeScript)
**Output:** `dist/nx-set-shas.js` (JavaScript)
**Bundler:** Bun
**Target:** Node.js runtime

**Commit Hook Integration:**

- Pre-commit hook runs build before allowing commit
- Compiled `dist/` files automatically added to commit if modified
- Ensures distributed version always matches source

## Format Validation

**Tool:** Prettier 3.6.2

**Check Command:**

```bash
prettier --check .
```

**Auto-format Command:**

```bash
prettier --write .
```

**Configuration:** `.prettierrc`

- Single quotes
- LF line endings
- 2-space indentation

**Lint-staged Integration:**

```json
"lint-staged": {
  "*.{ts,json,yml,md}": ["npx prettier --write"]
}
```

## Test Types

**Integration Tests:** Primary approach

- Real GitHub Actions environment
- Real git operations
- Real API calls to GitHub

**No Unit Tests:**

- No Jest/Vitest configuration
- Code structure designed to be directly executable
- Testing focuses on end-to-end workflow validation

## Common Patterns

**Async Testing:**

- GitHub Actions workflows handle async operations natively
- Workflow steps execute sequentially
- Environment variable outputs passed between steps via `${{ env.VAR_NAME }}`

**Error Assertion:**

```bash
# Pattern: Conditional throw
node -e "if(condition) console.log('success'); else { throw new Error('message');}"
```

**Command Output Capture:**

```bash
# Pattern: Store and verify
BASE_SHA=$(echo $(git merge-base origin/${{github.base_ref}} HEAD))
```

---

_Testing analysis: 2026-03-17_
