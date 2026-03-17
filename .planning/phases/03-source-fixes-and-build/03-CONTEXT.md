# Phase 3: Source Fixes and Build - Context

**Gathered:** 2026-03-17
**Status:** Ready for planning

<domain>
## Phase Boundary

Fix compiler-surfaced issues, audit all Node.js API usage for Node.js 24 compatibility, upgrade `@actions/*` dependencies to ESM-compatible versions with official Node 24 support, and rebuild the `dist/` artifact. Push and verify CI green before Phase 4.

</domain>

<decisions>
## Implementation Decisions

### TS2307 resolution strategy

- The `import { GitHub } from '@actions/github/lib/utils'` error (TS2307) is resolved by upgrading `@actions/github` to 9.0.0
- `@actions/github@9.0.0` explicitly exports `./lib/utils` in its `exports` map -- the original import works as-is
- No source code changes needed for the type import
- If `catch (e)` typing errors surface after the upgrade, fix them in Phase 3 (type guard or assertion)

### @actions/\* dependency upgrade

- Bump `@actions/core` from ^1.11.1 to ^3.0.0 (2.0.0 adds explicit Node 24 support; 3.0.0 is ESM-only)
- Bump `@actions/github` from ^6.0.1 to ^9.0.0 (ESM-only, fixes TypeScript compilation, updated octokit)
- Both packages are now ESM-only, aligning with the project's `"type": "module"`
- Verify API compatibility during planning: we use `getInput`, `getBooleanInput`, `setOutput`, `setFailed`, `exportVariable`, `getOctokit`, `context`, and `octokit.request`
- Verify `bun run build` succeeds after the upgrade (ESM deps + Bun bundler)
- This promotes DEPD-01 and DEPD-02 from v2 deferred requirements to v1 Phase 3 scope

### Audit scope and format

- API-by-API checklist table: each Node.js built-in API used, file and line numbers, Node.js 24 status, action needed
- Include deprecation research from Node.js 24 changelog
- Node.js built-in APIs only as primary scope (spawnSync, execSync, existsSync, process.\*)
- @actions/\* packages validated by Phase 1 baseline testing -- not re-audited here
- Note pre-commit hook `npm run build` vs `bun run build` discrepancy as a finding, but do not fix
- Document in `.planning/phases/03-source-fixes-and-build/03-AUDIT.md`
- Separate docs commit from code changes

### Build validation

- Validation sequence: `bun run build` -> `tsc --noEmit` -> review `git diff dist/` -> commit
- Push to remote and verify CI green within Phase 3
- Phase 3 owns CI green -- if tests fail, diagnose and fix within this phase

### Out-of-scope items (confirmed)

- `/dev/null` git argument on line 105 -- leave as-is, it's a git CLI argument on Linux/macOS runners, not a Node.js path
- `stripNewLineEndings()` latent bug (SRCF-01) -- deferred to v2, not part of migration
- stdio normalization (SRCF-02) -- deferred to v2
- Pre-commit hook `npm` vs `bun` discrepancy -- note in audit, do not fix

### Commit ordering

- Commit 1: `docs(03): audit Node.js 24 API compatibility` -- 03-AUDIT.md
- Commit 2: `feat(03): bump @actions/core to 3.x and @actions/github to 9.x` -- package.json + bun.lock
- Commit 3: `build(03): rebuild dist/ for node24 runtime` -- dist/nx-set-shas.js
- Then: push and verify CI green
- General commit title style; reference TS error codes in commit body, not title

### Claude's Discretion

- Exact audit table format and depth of deprecation research
- How to verify CI results (polling, waiting, etc.)
- Fix approach for any unexpected tsc errors after the upgrade
- Whether to combine any commits if they are trivially small

</decisions>

<specifics>
## Specific Ideas

- Phase 1 decision carried forward: "Always take the latest major version, deliver a modern codebase to upstream"
- The `@actions/github@9.0.0` `exports` map explicitly includes `"./lib/utils"` -- this is a supported public API, not an internal path
- `@actions/core@2.0.0` release note: "Add support for Node 24" (PR #2110) -- 1.x was not officially Node 24-compatible even though baseline testing passed
- Pre-commit hook runs `npm run build` which delegates to `bun build` via package.json script -- works but is a toolchain mismatch worth noting

</specifics>

<code_context>

## Existing Code Insights

### Reusable Assets

- `nx-set-shas.ts`: Single source file, uses `@actions/core`, `@actions/github`, `child_process.spawnSync`, `fs.existsSync`
- `tools/pre-commit.ts`: Pre-commit hook using `child_process.execSync`, `yoctocolors`
- `dist/nx-set-shas.js`: Built artifact (Bun bundle), checked into repo

### Established Patterns

- Bun as bundler: `bun build ./nx-set-shas.ts --outdir ./dist --target node`
- TypeScript type-checking: `tsc --noEmit` (separate from build)
- Pre-commit hook: runs build + format on every commit, auto-stages dist/ changes
- Package manager: Bun (bun.lock, packageManager field)

### Integration Points

- `@actions/github/lib/utils` provides `GitHub` type used in `findExistingCommit()` and `commitExists()` parameter types
- `@actions/core` provides all GitHub Actions I/O: getInput, setOutput, setFailed, exportVariable
- `@actions/github` provides `getOctokit()` and `context` for GitHub API calls
- `dist/nx-set-shas.js` is the runtime artifact consumed by `action.yml`
- CI workflows (`test.yml`, `integration-test-workflow.yml`, `format.yml`) test the action via `uses: ./`

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope. The @actions/\* upgrade was promoted from deferred to active scope based on research showing direct Node.js 24 relevance.

</deferred>

---

_Phase: 03-source-fixes-and-build_
_Context gathered: 2026-03-17_
