# Features Research: Node.js 24 Migration

**Domain:** GitHub Action runtime migration (Node.js 20 -> 24)
**Researched:** 2026-03-17
**Confidence:** HIGH (official migration guides, npm registry data, source code analysis)

## Table Stakes (Must Do)

Features that must change or the action breaks on node24 runtime.

| Feature                                                      | Why Required                                                                                                                                                                                              | Complexity | Notes                                                                                                                                                                                                |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Update `action.yml` runtime to `node24`                      | `using: 'node20'` will stop working when GitHub removes Node.js 20 from runners (fall 2026). This is the core migration change.                                                                           | Low        | Line 45: change `node20` to `node24`                                                                                                                                                                 |
| Update `@types/node` from 20.x to 24.x                       | TypeScript compilation will produce incorrect type information. `spawnSync`, `process`, `fs` types may differ.                                                                                            | Low        | Dev dependency only, no runtime impact, but needed for type-safe development                                                                                                                         |
| Update Volta/engines Node.js version                         | Development environment must match target runtime. Currently pinned to Node.js 20.19.4.                                                                                                                   | Low        | Update `volta.node` in package.json and `engines.node`                                                                                                                                               |
| Update `tsconfig.json` target/lib                            | Currently targeting ES2023. Node.js 24 supports ES2024+. Not strictly breaking, but `@types/node` 24.x may reference newer lib features.                                                                  | Low        | Update `lib` and `target` to at least ES2024                                                                                                                                                         |
| Version bump to 5.0.0                                        | Runtime change is a breaking change for consumers. Consumers must update their workflow references from `v4` to `v5`.                                                                                     | Low        | package.json version field                                                                                                                                                                           |
| Verify `spawnSync` behavior on Node.js 24                    | The action's core logic depends on `child_process.spawnSync` for all Git operations (rev-parse, merge-base, cat-file, hash-object). No known breaking changes, but this is the most critical API surface. | Low        | 5 distinct spawnSync call sites in `nx-set-shas.ts`. API is stable across Node.js 20-24. **HIGH confidence** no changes needed.                                                                      |
| Verify `process.stdout.write` behavior                       | Used extensively for logging (15+ call sites). No known breaking changes, but streams had behavioral changes in Node.js 24 (stricter error throwing).                                                     | Low        | **HIGH confidence** no changes needed for stdout.write to a TTY/pipe.                                                                                                                                |
| Verify `@actions/core` 1.11.1 compatibility with Node.js 24  | The current version uses `crypto.randomUUID()` (in bundled dist), `fs.existsSync`, `fs.appendFileSync`. These APIs are stable in Node.js 24.                                                              | Low        | **MEDIUM confidence.** The 1.x line should work on Node.js 24 -- it uses stable Node.js APIs. However, the latest version is 3.0.0 and the actions ecosystem is moving forward. See differentiators. |
| Verify `@actions/github` 6.0.1 compatibility with Node.js 24 | Uses Octokit for REST API calls. Depends on `undici@^5.28.5` which bundles its own HTTP client.                                                                                                           | Medium     | **MEDIUM confidence.** Undici 5.x was designed for Node.js 18-20. It may work on Node.js 24 but could hit subtle issues with the newer HTTP stack. The bundled dist includes a full copy of Undici.  |
| Test `existsSync` with Node.js 24                            | Used at line 30 for working directory validation. Node.js 24 adds runtime warnings for invalid inputs to `fs.existsSync()`.                                                                               | Low        | Only called with a string input, so no issue expected.                                                                                                                                               |

## Differentiators (Could Do)

Improvements enabled by Node.js 24 but not required for the migration to work.

| Feature                                                        | Value Proposition                                                                                                                                        | Complexity | Notes                                                                                                                                                                                                                                                        |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Upgrade `@actions/core` to 3.0.0                               | Latest version with updated dependencies (`@actions/exec@^3.0.0`, `@actions/http-client@^4.0.0`). Ensures long-term compatibility and security patches.  | Medium     | **Major version jump** (1.11.1 -> 3.0.0). API surface appears compatible (same exports: `getInput`, `setOutput`, `setFailed`, `exportVariable`, `getBooleanInput`). Needs testing. Risk: subtle behavioral changes in output/variable setting.               |
| Upgrade `@actions/github` to 9.0.0                             | Latest version uses `undici@^6.23.0`, `@octokit/core@^7.0.6`. Better Node.js 24 compatibility.                                                           | High       | **Major version jump** (6.0.1 -> 9.0.0). Octokit core went from v5 to v7. API for `github.getOctokit()` and `octokit.request()` likely changed. The `GitHub` type import from `@actions/github/lib/utils` may have moved. **HIGH risk of breaking changes.** |
| Update TypeScript target to ES2025                             | Node.js 24 (V8 13.6) supports ES2025 features natively. Could use Set methods, RegExp features, etc.                                                     | Low        | No benefit for this codebase -- it does not use any ES2025 features. Pure overhead.                                                                                                                                                                          |
| Replace `process.stdout.write` with `core.info`/`core.warning` | Better integration with GitHub Actions log grouping and annotations. `process.stdout.write` works but produces plain text without structured log levels. | Medium     | Would improve log readability in Actions UI. Not required for migration. Could be done incrementally.                                                                                                                                                        |
| Use `node:` protocol for all imports                           | Node.js 24 continues the trend of preferring `node:` prefixed imports. The source already uses `node:child_process` and `node:fs`.                       | Low        | Already done in source. The bundled dependencies in dist/ use unprefixed `require("fs")` etc., but those are third-party code and work fine.                                                                                                                 |

## Anti-Features (Don't Do)

Things to deliberately NOT change during this migration.

| Anti-Feature                                        | Why Avoid                                                                                                                                                                                                                                                                                                                              | What to Do Instead                                                                                                                                          |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Upgrade `@actions/github` to 9.0.0 during migration | 3 major versions of Octokit changes (core v5->v7). High risk of breaking API changes to `octokit.request()`, `github.getOctokit()`, and the `GitHub` type import. This is a separate refactoring effort, not a runtime migration.                                                                                                      | Keep `@actions/github@6.0.1`. It uses stable Node.js APIs and Undici 5.x which should work on Node.js 24. If compatibility issues arise, upgrade minimally. |
| Upgrade `@actions/core` to 3.0.0 during migration   | Two major version jumps. The 1.11.1 API surface is stable and uses basic Node.js APIs (`fs`, `os`, `crypto`). Upgrading adds risk with no clear necessity.                                                                                                                                                                             | Keep `@actions/core@1.11.1` unless testing reveals a Node.js 24 incompatibility.                                                                            |
| Refactor the monolithic `nx-set-shas.ts`            | Out of scope per PROJECT.md. The single-file architecture works and is not affected by the runtime change.                                                                                                                                                                                                                             | Leave architecture as-is.                                                                                                                                   |
| Switch build toolchain from Bun                     | Bun bundler is working and not related to the Node.js 24 runtime. The action runs on Node.js, not Bun.                                                                                                                                                                                                                                 | Keep `bun build` as-is.                                                                                                                                     |
| Add backwards compatibility for node20              | PROJECT.md explicitly scopes this as a clean v5 break. Dual runtime support adds complexity with no benefit -- GitHub is removing Node.js 20 anyway.                                                                                                                                                                                   | Single `node24` target.                                                                                                                                     |
| Update Prettier, Husky, lint-staged                 | Dev tooling unrelated to the Node.js 24 runtime. These run in the dev environment, not in GitHub Actions runners.                                                                                                                                                                                                                      | Leave at current versions.                                                                                                                                  |
| Modernize Octokit API calls                         | The action uses raw `octokit.request('GET /repos/...')` calls. While Octokit has typed endpoint methods, changing the call pattern is refactoring, not migration.                                                                                                                                                                      | Keep existing `octokit.request()` pattern.                                                                                                                  |
| Fix the `/dev/null` usage on Linux runners          | Line 105 uses `git hash-object -t tree /dev/null` to get the empty tree hash. This works correctly on Linux (where GitHub Actions runners run) because `/dev/null` exists. It would fail on Windows, but Actions runners are Linux/macOS. This is NOT a Node.js 24 issue.                                                              | Leave as-is. This is a Git command argument, not a Node.js file path.                                                                                       |
| Change `stripNewLineEndings` to use `replaceAll`    | The current `string.replace('\n', '')` only replaces the first newline. `replaceAll` is available since Node.js 15. While technically a bug (if multiple newlines exist), this has worked correctly in production because `spawnSync` output has exactly one trailing newline. Fixing this is a behavior change, not a migration task. | Leave as-is or defer to a separate fix.                                                                                                                     |

## Feature Dependencies

```
action.yml node24 runtime
  --> @types/node 24.x (for correct type checking)
  --> tsconfig.json target update (for lib compatibility with @types/node 24.x)
  --> Volta/engines Node.js 24 (for local dev matching runtime)
  --> Version bump to 5.0.0 (required by breaking runtime change)
  --> Rebuild dist/ with Bun (to produce node24-compatible bundle)

All verification tasks (spawnSync, stdout, @actions/core, @actions/github)
  --> Can be done in parallel
  --> Must complete before release

@actions/core upgrade (if needed)
  --> Would require @actions/github upgrade (shared http-client dep)
  --> Would require full test pass
  --> DO NOT chain upgrades -- only upgrade if Node.js 24 breaks 1.11.1
```

## MVP Recommendation

**Minimal migration path (recommended):**

1. Update `action.yml` from `node20` to `node24`
2. Update `@types/node` from 20.x to 24.x
3. Update `tsconfig.json` target/lib to ES2024
4. Update Volta pin to Node.js 24.x LTS
5. Update `engines.node` to `>=24`
6. Bump version to 5.0.0
7. Rebuild `dist/` and run full test suite
8. Test with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` in a real workflow

**Defer:** `@actions/core` and `@actions/github` upgrades unless testing reveals incompatibilities.

**Rationale:** The codebase uses only stable, unchanged Node.js APIs (`child_process.spawnSync`, `process.stdout.write`, `fs.existsSync`, `process.chdir`, `process.env`). None of these have breaking changes between Node.js 20 and 24. The highest risk is in the bundled third-party code (Undici 5.x in `@actions/github`), which should be validated through integration testing rather than preemptive upgrades.

## Sources

- [Node.js v20 to v22 Migration Guide](https://nodejs.org/en/blog/migrations/v20-to-v22) -- Official
- [Node.js v22 to v24 Migration Guide](https://nodejs.org/en/blog/migrations/v22-to-v24) -- Official
- [Node.js 24.0.0 Release Notes](https://nodejs.org/en/blog/release/v24.0.0) -- Official
- [GitHub Deprecation of Node 20 on Actions Runners](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/) -- Official
- [GitHub Actions Node.js Plan Discussion](https://github.com/orgs/community/discussions/160454) -- Community
- [actions/toolkit Repository](https://github.com/actions/toolkit) -- Official
- [@actions/core npm Registry](https://www.npmjs.com/package/@actions/core) -- v3.0.0 latest
- [@actions/github npm Registry](https://www.npmjs.com/package/@actions/github) -- v9.0.0 latest
- Source code analysis of `nx-set-shas.ts` and `dist/nx-set-shas.js` -- Direct
