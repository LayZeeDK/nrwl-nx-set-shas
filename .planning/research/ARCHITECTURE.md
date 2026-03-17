# Architecture Research: Node.js 24 Migration

**Project:** nx-set-shas
**Researched:** 2026-03-17
**Overall confidence:** HIGH

## Files Requiring Changes

### Tier 1: Runtime Declaration (must change first)

| File                   | Change                                 | Why                                                                                                                                          |
| ---------------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `action.yml` (line 45) | `using: 'node20'` -> `using: 'node24'` | Core runtime declaration. GitHub runners will stop supporting `node20` actions after June 2, 2026. This is the single most important change. |

### Tier 2: Type Definitions (must change before auditing source)

| File                     | Change                                     | Why                                                                                                                                                   |
| ------------------------ | ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `package.json` (line 29) | `@types/node` from `^20.19.9` to `^24.0.0` | TypeScript type definitions must match target runtime. Updating types first surfaces API incompatibilities at compile time, guiding the source audit. |

### Tier 3: Source Code Audit

| File                  | Lines              | API Used                                                       | Risk   | Change Needed                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| --------------------- | ------------------ | -------------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nx-set-shas.ts`      | 4                  | `spawnSync` from `node:child_process`                          | LOW    | No breaking changes to `spawnSync` between Node 20 and 24 for non-`.bat`/`.cmd` executables. This code spawns `git` directly (not a `.cmd` file), so the EINVAL security fix (CVE-2024-27980) does not apply. No change required.                                                                                                                                                                                                              |
| `nx-set-shas.ts`      | 5                  | `existsSync` from `node:fs`                                    | LOW    | No breaking changes. Node 24 added stricter type validation for arguments, but this code passes a string (from `core.getInput`), which is correct. No change required.                                                                                                                                                                                                                                                                         |
| `nx-set-shas.ts`      | 31                 | `process.chdir()`                                              | NONE   | Stable API, no changes between Node 20 and 24.                                                                                                                                                                                                                                                                                                                                                                                                 |
| `nx-set-shas.ts`      | 33-34, 87-88, etc. | `process.stdout.write()`                                       | NONE   | Stable API, no changes.                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `nx-set-shas.ts`      | 12                 | `process.env` assignment                                       | NONE   | Stable API.                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `nx-set-shas.ts`      | 103-106            | `spawnSync('git', ['hash-object', '-t', 'tree', '/dev/null'])` | MEDIUM | **Pre-existing issue, not Node 24 related**: `/dev/null` is Unix-specific. On Windows runners, Git for Windows translates this via MSYS2 path conversion, so it works in Git Bash context. However, when `spawnSync` spawns `git` directly (not through a shell), MSYS2 path translation does NOT occur. This has always been a potential issue on Windows but is outside migration scope (not a Node 24 regression). Flag for awareness only. |
| `nx-set-shas.ts`      | 280-282            | `string.replace('\n', '')`                                     | NONE   | Pure JavaScript, not a Node.js API. No change.                                                                                                                                                                                                                                                                                                                                                                                                 |
| `tools/pre-commit.ts` | 1                  | `execSync` from `node:child_process`                           | LOW    | `execSync` spawns `npm run build` and `npm run format`. On Windows, npm resolves to `npm.cmd`, which requires `shell: true` in Node 24 to avoid EINVAL. However, `execSync` **always runs in a shell by default**, so this is safe. No change required.                                                                                                                                                                                        |
| `tools/pre-commit.ts` | 8, 41              | `process.exit()`                                               | NONE   | Stable API.                                                                                                                                                                                                                                                                                                                                                                                                                                    |

### Tier 4: Configuration Updates

| File                     | Change                                                                     | Why                                                                                                                                                        |
| ------------------------ | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `package.json` (line 4)  | Version `4.4.0` -> `5.0.0`                                                 | Major version bump signals breaking change (node24 runtime requirement).                                                                                   |
| `package.json` (line 17) | `engines.node` from `>=20` to `>=24`                                       | Reflects new minimum Node.js version for development.                                                                                                      |
| `package.json` (line 20) | `volta.node` from `20.19.4` to latest Node 24 LTS                          | Pins development Node.js version via Volta.                                                                                                                |
| `tsconfig.json`          | Consider updating `target` and `lib` from `ES2023` to `ES2024` or `ES2025` | Node 24 (V8 13.6) supports ES2025 features. Optional -- ES2023 target still works fine on Node 24. Only update if you want to use newer language features. |

### Tier 5: Build Output Regeneration

| File                  | Change                      | Why                                                                                                                                           |
| --------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `dist/nx-set-shas.js` | Rebuild via `bun run build` | Build artifact must be regenerated after any source or dependency changes. This file is committed to the repo and referenced by `action.yml`. |

### Tier 6: Dependencies (conditional)

| Package           | Current                                   | Action                                   | Rationale                                                                                                                                                                                            |
| ----------------- | ----------------------------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@actions/core`   | 1.11.1                                    | **Evaluate update to ^1.11.1 or ^3.0.0** | Current version 1.11.1 should work on Node 24 (pure JS, no native addons). Latest is 3.0.0. Update only if compatibility testing reveals issues. Per project constraint: minimal dependency updates. |
| `@actions/github` | 6.0.1                                     | **Keep as-is unless issues found**       | Pure JS package. Transitive dep on `undici@5.29.0` -- this is the main risk area since Undici is sensitive to Node.js version. Test first.                                                           |
| `undici`          | 5.29.0 (transitive via `@actions/github`) | **Monitor**                              | `@actions/github@6.0.1` pins `undici@^5.28.5`. Node 24 ships with built-in `undici@7`. The bundled v5 should still work but is the most likely source of subtle incompatibilities.                   |

## Migration Sequence

The correct order of operations, with explicit dependencies:

```
Phase 1: Foundation (no code changes, config only)
  1.1  action.yml: node20 -> node24
  1.2  package.json: volta.node -> Node 24 LTS
  1.3  package.json: engines.node -> >=24

Phase 2: Type System (enables compile-time verification)
  2.1  package.json: @types/node -> ^24.0.0
  2.2  bun install (update lockfile)
  2.3  tsc --noEmit (verify compilation with new types)
       -> Fix any type errors surfaced

Phase 3: Source Audit (informed by type errors from Phase 2)
  3.1  Audit nx-set-shas.ts for deprecated/removed Node.js APIs
       -> Based on research: NO source changes expected
  3.2  Audit tools/pre-commit.ts
       -> Based on research: NO source changes expected

Phase 4: Build and Test
  4.1  bun run build (regenerate dist/nx-set-shas.js)
  4.2  Run test workflows with FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true
  4.3  Verify on all three OS matrices (ubuntu, macos, windows)

Phase 5: Version Bump
  5.1  package.json: version 4.4.0 -> 5.0.0
  5.2  Optional: tsconfig.json target/lib -> ES2024 or ES2025
```

### Dependency Graph

```
action.yml ─────────────────────────────────────────> dist/nx-set-shas.js
                                                          ^
package.json (volta, engines) ──> bun install ──> @types/node@24
                                                          |
                                                    tsc --noEmit
                                                          |
                                                    source audit
                                                          |
                                                    bun run build ──> dist/nx-set-shas.js
```

Key insight: `action.yml` and `package.json` config changes are independent of each other and can be done in any order. But `@types/node` must be updated BEFORE the source audit, because the new type definitions will surface any API incompatibilities at compile time.

## Build Pipeline Impact

### Current Build Pipeline

```
nx-set-shas.ts ──[bun build]──> dist/nx-set-shas.js ──[git commit]──> action.yml references it
```

### Impact Assessment

| Pipeline Step     | Impact | Details                                                                                                                                                                                                                                       |
| ----------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bun build`       | NONE   | Bun is the build tool, not the runtime. Bun bundles TypeScript to a single JS file targeting Node. The `--target node` flag in the build command does not specify a Node version -- it produces generic Node-compatible JS. No change needed. |
| `bun install`     | LOW    | Lockfile will update when `@types/node` version changes.                                                                                                                                                                                      |
| `tsc --noEmit`    | MEDIUM | New `@types/node@24` may surface type errors for removed/changed APIs. This is the primary verification mechanism.                                                                                                                            |
| Pre-commit hook   | NONE   | `tools/pre-commit.ts` runs under Bun (not Node), so Node version is irrelevant for the hook itself. It calls `npm run build` which invokes `bun build`.                                                                                       |
| CI test workflows | MEDIUM | Tests use `uses: ./` to run the action, which means the runner's Node version matters. Need `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` to test before June 2026 deadline.                                                                      |

### Build Artifact Size

No significant size change expected. The action bundles `@actions/core`, `@actions/github`, and their transitive dependencies (including `undici@5`). Node 24 ships with `undici@7` built-in, but since the action bundles its own copy, there is no size change from the runtime upgrade.

## Testing Strategy

### Pre-Migration Verification

Before making any changes, establish a baseline:

1. Run existing CI on current `node20` to confirm all tests pass (baseline)
2. Run existing CI with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` env to see if the current code already works on Node 24 without changes

### Migration Verification

After making changes:

| Test                    | How                                                                     | What It Validates                |
| ----------------------- | ----------------------------------------------------------------------- | -------------------------------- |
| Type check              | `tsc --noEmit` with `@types/node@24`                                    | No removed/changed APIs used     |
| Build                   | `bun run build`                                                         | TypeScript compiles to valid JS  |
| Unit-level (PR event)   | `test.yml` PR workflow with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true`   | merge-base calculation works     |
| Unit-level (push event) | `test.yml` push workflow with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` | Successful workflow lookup works |
| Integration             | `test-integration.yml` with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true`   | Working directory support works  |
| Cross-platform          | All three OS matrices (ubuntu, macos, windows)                          | No platform-specific regressions |

### CI Workflow Changes for Testing

Add `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` as a job-level environment variable in test workflows during migration. This forces the runner to execute the action with Node 24 even though runners default to Node 20 until June 2026.

```yaml
jobs:
  test:
    env:
      FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
```

After June 2, 2026, this env var becomes unnecessary (Node 24 will be the default).

### Risk Areas Ranked

1. **`undici@5` compatibility with Node 24** (MEDIUM) -- The transitive `undici@5.29.0` from `@actions/github` is the most likely failure point. Node 24 ships `undici@7` and internal HTTP handling changed. The bundled v5 should still work as a standalone module, but HTTP/fetch behavior edge cases are possible.
2. **Windows `/dev/null` in `git hash-object`** (LOW, pre-existing) -- Not a Node 24 regression but worth noting. This code path only triggers when no successful workflow is found AND HEAD~1 does not exist (rare edge case on brand-new repos).
3. **`@types/node@24` type compatibility** (LOW) -- May surface minor type signature changes requiring adjustments, but unlikely given the APIs used are all stable.

## Sources

- [Node.js v22 to v24 Migration Guide](https://nodejs.org/en/blog/migrations/v22-to-v24) -- HIGH confidence, official
- [Deprecation of Node 20 on GitHub Actions runners](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/) -- HIGH confidence, official
- [Node.js v20 to v22 Migration Guide](https://nodejs.org/en/blog/migrations/v20-to-v22) -- HIGH confidence, official
- [Node.js 24 Becomes LTS](https://nodesource.com/blog/nodejs-24-becomes-lts) -- MEDIUM confidence
- [actions/setup-node Node 24 PR](https://github.com/actions/setup-node/pull/1325) -- MEDIUM confidence, shows pattern for migration
- [actions/toolkit RELEASES.md](https://github.com/actions/toolkit/blob/main/packages/core/RELEASES.md) -- HIGH confidence, official
- [FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 runner issue](https://github.com/actions/runner/issues/4295) -- MEDIUM confidence, confirms testing mechanism
