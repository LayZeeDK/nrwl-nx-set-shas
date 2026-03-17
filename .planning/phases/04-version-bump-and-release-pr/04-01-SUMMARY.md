---
phase: 04-version-bump-and-release-pr
plan: 01
subsystem: infra
tags: [github-actions, node24, cherry-pick, release, pr]

# Dependency graph
requires:
  - phase: 03-source-fixes-and-build
    provides: dist/nx-set-shas.js rebuilt with @actions/core@3.x and @actions/github@9.x
provides:
  - Clean branch feat/node24-runtime on origin with 4 commits (3 migration + 1 version bump)
  - package.json version bumped to 5.0.0
  - Open upstream PR nrwl/nx-set-shas#210
affects: []

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Cherry-pick migration commits onto a clean branch from upstream/main to avoid polluting upstream PR with GSD artifacts
    - Pre-commit hook auto-rebuilds and stages dist/ on version bump commit

key-files:
  created: []
  modified:
    - package.json

key-decisions:
  - 'One cherry-pick (chore: remove FORCE_JAVASCRIPT_ACTIONS_TO_NODE24) was skipped because it was already in upstream/main -- clean branch has 4 commits (3 migration + 1 version bump) instead of planned 5'
  - 'dist/ conflict in cherry-pick of feat(03) resolved by taking theirs (the migration version) -- correct as it was built with upgraded @actions/* deps'
  - 'PR opened as nrwl/nx-set-shas#210 targeting main with feat! conventional commit title'

patterns-established:
  - 'PR to upstream is the release mechanism -- no manual tagging needed (publish.yml applies v5/v5.0/v5.0.0 tags after merge)'

requirements-completed: [RLSE-01, RLSE-02, RLSE-03, RLSE-04]

# Metrics
duration: 15min
completed: 2026-03-17
---

# Phase 4 Plan 01: Version Bump and Release PR Summary

**Node.js 24 migration packaged as v5.0.0 on a clean 4-commit branch and submitted as PR nrwl/nx-set-shas#210 targeting upstream main**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-03-17T21:30:00Z
- **Completed:** 2026-03-17T21:45:00Z
- **Tasks:** 5 (Tasks 1-3 auto, Task 4 CI verified green automatically, Task 5 PR created)
- **Files modified:** 1 (package.json on feat/node24-runtime)

## Accomplishments

- Created clean branch `feat/node24-runtime` from `upstream/main` with 3 cherry-picked migration commits
- Bumped `package.json` version to 5.0.0 via pre-commit hook (which also rebuilt and validated `dist/`)
- Pushed branch to origin, triggered CI (both `test.yml` and `test-integration.yml` passed green)
- Opened upstream PR https://github.com/nrwl/nx-set-shas/pull/210 with full breaking change documentation

## Task Commits (on feat/node24-runtime)

Tasks 1-3 created commits on the clean branch `feat/node24-runtime`:

1. **Task 1: Cherry-pick migration commits**
   - `2326a8b` feat(02-01): update runtime and toolchain to Node.js 24 / ES2024
   - `490fea5` feat(02-02): bump @types/node from ^20.19.9 to ^24.12.0
   - `d67edfe` feat(03): bump @actions/core to 3.x and @actions/github to 9.x

2. **Task 2: Bump version to 5.0.0**
   - `e88981b` chore: bump version to 5.0.0

3. **Task 3: Push to origin** (no new commit -- git push only)

4. **Task 4: CI verified green** (no commit -- verification only)

5. **Task 5: Open upstream PR** (no commit -- gh CLI PR creation only)

**Working branch metadata:** `2895f4d` chore(planning): add \_auto_chain_active to config

## Files Created/Modified

- `package.json` (on feat/node24-runtime) -- version bumped from 4.4.0 to 5.0.0

## Decisions Made

- Skipped the second cherry-pick (`chore(02-01): remove FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`) because it was already in `upstream/main` -- resulted in 3 migration commits instead of 4, for a total of 4 commits (3 + version bump)
- Resolved `dist/nx-set-shas.js` merge conflict during cherry-pick of `feat(03)` by taking `--theirs` (the migration version built with upgraded @actions/\* packages)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] One cherry-pick was empty (already applied upstream)**

- **Found during:** Task 1 (Create clean branch and cherry-pick migration commits)
- **Issue:** `chore(02-01): remove FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` cherry-pick produced an empty commit because the change was already incorporated in `upstream/main`
- **Fix:** Used `git cherry-pick --skip` to skip the empty commit. Clean branch has 4 commits instead of planned 5 -- all migration intent is preserved.
- **Files modified:** None (correctly skipped)
- **Verification:** `git log --oneline upstream/main..HEAD` shows exactly 4 meaningful commits
- **Committed in:** N/A (skipped)

**2. [Rule 1 - Bug] dist/nx-set-shas.js merge conflict during cherry-pick**

- **Found during:** Task 1 (cherry-pick of feat(03))
- **Issue:** The dist/ bundle from upstream/main diverged from the bundle in the migration commit (different @actions/\* versions produce different bundles)
- **Fix:** Resolved by taking `--theirs` (the migration branch version) which was built with @actions/core@3.x and @actions/github@9.x
- **Files modified:** dist/nx-set-shas.js
- **Verification:** Pre-commit hook ran during cherry-pick --continue and confirmed build successful
- **Committed in:** d67edfe (part of cherry-pick continuation)

---

**Total deviations:** 2 auto-fixed (both Rule 1 - Bug)
**Impact on plan:** Both deviations handled correctly without compromising migration intent. The final branch is cleaner than planned (no redundant empty commit).

## Issues Encountered

- `git checkout -b feat/node24-runtime upstream/main` failed initially because `.planning/config.json` had an unstaged change (GSD tooling had added `_auto_chain_active` field). Resolved by stashing the file, creating the branch, then restoring and committing the stash on the working branch.

## CI Results

Both fork CI workflows passed green on `feat/node24-runtime`:

- [test.yml](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23217767054) -- ubuntu-latest, macOS-latest, windows-latest (conclusion: success)
- [test-integration.yml](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23217767087) -- ubuntu-latest, macOS-latest, windows-latest (conclusion: success)

`format.yml` and `publish.yml` will run on PR events (visible in PR Checks tab).

## PR Details

- **URL:** https://github.com/nrwl/nx-set-shas/pull/210
- **Title:** `feat!: update action runtime to node24`
- **Base:** `nrwl/nx-set-shas:main`
- **Head:** `LayZeeDK:feat/node24-runtime`
- **State:** OPEN
- **Closes:** nrwl/nx-set-shas#208

## User Setup Required

None - PR is open and CI is green. No external service configuration required beyond reviewing the upstream PR.

## Next Phase Readiness

Phase 4 is complete. The PR is open and awaiting upstream maintainer review.

- Working branch `LayZeeDK/feat/migrate-to-node24-runtime` is preserved as a permanent record of the full migration process including GSD planning artifacts
- Upon merge, `publish.yml` will automatically tag `v5`, `v5.0`, and `v5.0.0`

---

_Phase: 04-version-bump-and-release-pr_
_Completed: 2026-03-17_
