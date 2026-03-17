---
phase: 02-configuration-and-type-system
plan: 01
subsystem: infra
tags: [node24, es2024, github-actions, typescript, volta]

# Dependency graph
requires:
  - phase: 01-baseline-validation
    provides: Confirmed action works on Node.js 24 with zero source changes
provides:
  - Node.js 24 runtime declaration in action.yml
  - ES2024 TypeScript compiler target and lib
  - Node.js 24 toolchain pins (Volta, engines)
  - Clean CI workflows without FORCE flag workaround
affects: [02-configuration-and-type-system, 03-build-system, 04-ci-release]

# Tech tracking
tech-stack:
  added: []
  patterns: [node24-native-runtime, es2024-target]

key-files:
  created: []
  modified:
    - action.yml
    - package.json
    - tsconfig.json
    - .github/workflows/test.yml
    - .github/workflows/test-integration.yml
    - .github/workflows/format.yml
    - .github/workflows/publish.yml
    - .github/workflows/integration-test-workflow.yml

key-decisions:
  - 'ES2024 target and lib per official Node.js Target Mapping for Node 24'
  - 'module remains nodenext -- locked decision from planning phase'

patterns-established:
  - 'node24 native runtime: action.yml using field declares node24 directly'

requirements-completed: [RUNT-01, RUNT-02, RUNT-03, TSCO-01, TSCO-02]

# Metrics
duration: 2min
completed: 2026-03-17
---

# Phase 2 Plan 1: Runtime and Toolchain Configuration Summary

**Node.js 24 runtime declaration with ES2024 TypeScript targets and FORCE flag cleanup across all CI workflows**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-17T14:22:22Z
- **Completed:** 2026-03-17T14:24:08Z
- **Tasks:** 2
- **Files modified:** 8

## Accomplishments

- action.yml now declares node24 natively -- GitHub Actions runners will use Node.js 24 without workarounds
- TypeScript compiler targets updated to ES2024 per official Node.js Target Mapping
- Volta pin and engines field updated to Node.js 24.14.0 / >=24
- FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 removed from all 5 CI workflow files

## Task Commits

Each task was committed atomically:

1. **Task 1: Update runtime declaration and toolchain config** - `f7ea1ff` (feat)
2. **Task 2: Remove FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 from all CI workflows** - `c3f3ceb` (chore)

## Files Created/Modified

- `action.yml` - Runtime declaration changed from node20 to node24
- `package.json` - engines >=24, volta pin 24.14.0
- `tsconfig.json` - target and lib both ES2024
- `.github/workflows/test.yml` - Removed FORCE flag env block
- `.github/workflows/test-integration.yml` - Removed FORCE flag env block
- `.github/workflows/format.yml` - Removed FORCE flag env block
- `.github/workflows/publish.yml` - Removed FORCE flag env block
- `.github/workflows/integration-test-workflow.yml` - Removed FORCE flag env block

## Decisions Made

- ES2024 for both target and lib per official Node.js Target Mapping for Node.js 24
- module remains nodenext (locked decision from Phase 2 planning context)

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- All config files updated -- ready for Plan 02 (@types/node upgrade) and Plan 03 (TypeScript strict settings)
- Build system (Phase 3) can proceed with ES2024 targets in place

---

_Phase: 02-configuration-and-type-system_
_Completed: 2026-03-17_
