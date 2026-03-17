---
phase: 03-source-fixes-and-build
plan: 01
subsystem: api-compatibility
tags: [node24, actions-core, actions-github, audit, esm]

# Dependency graph
requires:
  - phase: 02-config-and-types
    provides: tsconfig.json ES2024 target/lib, @types/node 24.x
provides:
  - Node.js 24 API compatibility audit document
  - @actions/core 3.x and @actions/github 9.x ESM-only dependencies
  - Zero tsc errors on source files
affects: [03-source-fixes-and-build]

# Tech tracking
tech-stack:
  added: ["@actions/core@3.0.0", "@actions/github@9.0.0"]
  patterns: [skipLibCheck for third-party octokit type mismatches]

key-files:
  created: [".planning/phases/03-source-fixes-and-build/03-AUDIT.md"]
  modified: ["package.json", "bun.lock", "tsconfig.json", "dist/nx-set-shas.js"]

key-decisions:
  - "Added skipLibCheck to tsconfig.json to handle octokit internal type mismatches between @octokit/plugin-paginate-rest and @octokit/types"
  - "Zero Node.js API breaking changes confirmed -- no source modifications needed"

patterns-established:
  - "skipLibCheck: true is standard when third-party library types have internal conflicts"

requirements-completed: [AUDT-01, AUDT-02, AUDT-03]

# Metrics
duration: 2min
completed: 2026-03-17
---

# Phase 3 Plan 1: Audit and Dependency Upgrade Summary

**Node.js 24 API audit confirming zero breaking changes, plus @actions/core 3.x and @actions/github 9.x ESM-only upgrade with skipLibCheck for octokit type conflicts**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-17T17:26:38Z
- **Completed:** 2026-03-17T17:28:59Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments

- Audited all Node.js built-in APIs in both source files, confirming zero breaking changes for Node.js 24
- Upgraded @actions/core to 3.0.0 and @actions/github to 9.0.0 (ESM-only versions with Node 24 support)
- Resolved TS2307 error on `@actions/github/lib/utils` via the new exports map in @actions/github 9.0.0
- Added skipLibCheck to handle octokit internal type definition mismatches

## Task Commits

Each task was committed atomically:

1. **Task 1: Create Node.js 24 API compatibility audit document** - `0124c1e` (docs)
2. **Task 2: Upgrade @actions/core to 3.x and @actions/github to 9.x** - `54af1c8` (feat)

## Files Created/Modified

- `.planning/phases/03-source-fixes-and-build/03-AUDIT.md` - API-by-API compatibility audit for both source files
- `package.json` - @actions/core ^3.0.0 and @actions/github ^9.0.0
- `bun.lock` - Updated dependency tree with new octokit transitive deps
- `tsconfig.json` - Added skipLibCheck: true
- `dist/nx-set-shas.js` - Rebuilt by pre-commit hook with ESM-only @actions/\* deps

## Decisions Made

- **skipLibCheck: true** - Added to tsconfig.json because @actions/github 9.0.0 pulls in @octokit/plugin-paginate-rest and @octokit/plugin-rest-endpoint-methods which have internal type mismatches against @octokit/types. All errors were in node_modules/ only -- zero errors in source files. This is standard practice for projects that consume libraries with imperfect type alignment.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added skipLibCheck to tsconfig.json for octokit type conflicts**

- **Found during:** Task 2 (dependency upgrade)
- **Issue:** `tsc --noEmit` produced ~100 errors, all in `node_modules/@octokit/plugin-paginate-rest` and `node_modules/@octokit/plugin-rest-endpoint-methods` referencing endpoints missing from `@octokit/types/Endpoints`. Zero errors in source files.
- **Fix:** Added `"skipLibCheck": true` to tsconfig.json compilerOptions
- **Files modified:** tsconfig.json
- **Verification:** `npx tsc --noEmit` exits with code 0
- **Committed in:** 54af1c8 (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Necessary to achieve the "tsc --noEmit passes" done criterion. skipLibCheck is standard practice and does not affect source-level type safety.

## Issues Encountered

None beyond the octokit type mismatch handled above.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- API audit complete, dependencies upgraded, type checking passes
- Ready for Plan 2 (dist/ rebuild and CI verification) or remaining Phase 3 plans
- Pre-commit hook already rebuilt dist/ during the Task 2 commit, so dist/nx-set-shas.js reflects the new ESM-only @actions/\* deps

## Self-Check: PASSED

- [x] 03-AUDIT.md exists
- [x] 03-01-SUMMARY.md exists
- [x] Commit 0124c1e found (Task 1)
- [x] Commit 54af1c8 found (Task 2)

---

_Phase: 03-source-fixes-and-build_
_Completed: 2026-03-17_
