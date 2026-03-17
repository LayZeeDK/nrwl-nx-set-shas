# nx-set-shas Node.js 24 Migration

## What This Is

Migrate the `nrwl/nx-set-shas` GitHub Action from the deprecated Node.js 20 runtime to Node.js 24. This is a major version bump (v4 -> v5) that updates the action runtime, audits all Node.js API usage for breaking changes between Node.js 20/22/24, and ensures all consumers stop seeing deprecation warnings.

## Core Value

The action runs without deprecation warnings on GitHub Actions Node.js 24 runtime and all existing functionality works correctly.

## Requirements

### Validated

- v SHA resolution for pull_request events (merge-base calculation) -- existing
- v SHA resolution for merge_group events -- existing
- v SHA resolution for push/workflow_dispatch events (successful workflow lookup) -- existing
- v Fallback chain when no successful workflow found (user SHA -> HEAD~1 -> empty tree) -- existing
- v GitHub Actions inputs/outputs interface (action.yml) -- existing
- v GitHub API integration for workflow run queries -- existing
- v Git operations via spawnSync (rev-parse, merge-base, cat-file, hash-object) -- existing
- v Build pipeline (Bun build to dist/) -- existing
- v Pre-commit hook for build artifact validation -- existing

### Active

- [ ] Update action.yml runtime from node20 to node24
- [ ] Audit and fix all Node.js API usage for Node.js 24 compatibility
- [ ] Update @types/node from 20.x to 24.x
- [ ] Update dependencies only where required for Node.js 24 compatibility
- [ ] Bump package version from 4.4.0 to 5.0.0
- [ ] All existing tests pass on Node.js 24
- [ ] Update Volta/engine configuration for Node.js 24
- [ ] Document breaking changes for v5 consumers

### Out of Scope

- Build toolchain modernization (keep Bun bundler as-is) -- minimal scope, only change what's needed
- Dependency updates unrelated to Node.js 24 -- only update if compatibility requires it
- Backwards compatibility with node20 runtime -- clean v5 break, no dual support
- New features or refactoring -- migration only

## Context

- GitHub is deprecating Node.js 20 for Actions; forced migration to Node.js 24 on June 2nd, 2026
- All consumers of `nrwl/nx-set-shas@v4` currently see deprecation warnings
- The codebase is a single TypeScript file (`nx-set-shas.ts`) compiled via Bun to `dist/nx-set-shas.js`
- Current stack: TypeScript 5.8.3, Bun 1.2.19, Node.js 20.19.4 (Volta), @actions/core 1.11.1, @actions/github 6.0.1
- The action uses `child_process.spawnSync` for Git operations and Octokit for GitHub API calls
- Existing codebase map available at `.planning/codebase/`

## Constraints

- **Toolchain**: Keep Bun as bundler and package manager -- no build toolchain changes
- **Scope**: Only change what Node.js 24 migration requires -- no feature work
- **Dependencies**: Update deps only when they block Node.js 24 compatibility
- **Release**: Major version bump to v5 (breaking change)

## Key Decisions

| Decision                   | Rationale                                             | Outcome    |
| -------------------------- | ----------------------------------------------------- | ---------- |
| Major version bump (v5)    | Node.js 24 runtime is a breaking change for consumers | -- Pending |
| No backwards compatibility | Clean break, no dual node20/node24 support            | -- Pending |
| Minimal dependency updates | Reduce risk, only update what Node.js 24 requires     | -- Pending |
| Keep Bun bundler           | Working fine, not related to Node.js 24 migration     | -- Pending |

---

_Last updated: 2026-03-17 after initialization_
