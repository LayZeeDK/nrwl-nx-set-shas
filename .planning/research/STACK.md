# Stack Research: Node.js 24 Migration

**Project:** nrwl/nx-set-shas GitHub Action
**Researched:** 2026-03-17
**Focus:** What changes between Node.js 20 and 24 for a GitHub Action

## Node.js Version Changes (20 -> 22 -> 24)

GitHub Actions skips Node.js 22 entirely -- the runtime jumps from `node20` to `node24`. However, the Node.js breaking changes accumulate across both major versions. Here are the changes relevant to this codebase.

### Relevant Breaking Changes (20 -> 22)

| Change                                                        | Impact on This Project                                             | Action Required |
| ------------------------------------------------------------- | ------------------------------------------------------------------ | --------------- |
| Import assertions (`assert`) removed, `with` keyword required | **None** -- codebase does not use import assertions for JSON       | No              |
| Streams default high water mark: 16 KiB -> 64 KiB             | **Negligible** -- spawnSync buffers are not affected by stream HWM | No              |
| `require()` for ES modules available by default               | **None** -- project already uses ESM (`"type": "module"`)          | No              |

### Relevant Breaking Changes (22 -> 24)

| Change                                                               | Impact on This Project                                                                                         | Action Required |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------- |
| OpenSSL 3.5, security level 2 (RSA < 2048 bits prohibited)           | **None** -- action does not perform crypto operations; GitHub API uses HTTPS via Octokit which uses system TLS | No              |
| `fs.F_OK`, `fs.R_OK` etc. deprecated, use `fs.constants.*`           | **None** -- codebase uses `existsSync()` only, no access mode constants                                        | No              |
| `fs.truncate()` with fd deprecated                                   | **None** -- not used                                                                                           | No              |
| `dirent.path` deprecated, use `dirent.parentPath`                    | **None** -- not used                                                                                           | No              |
| `process.assert` removed                                             | **None** -- not used                                                                                           | No              |
| Stricter `Buffer.write()` beyond length throws                       | **None** -- codebase does not manipulate Buffers directly (spawnSync output is `.toString()`'d)                | No              |
| `Buffer.allocUnsafe` temporarily returned zero-filled in 24.11.0 LTS | **None** -- not used                                                                                           | No              |
| V8 upgraded to 13.6                                                  | **None** -- no native addons, pure JS/TS                                                                       | No              |
| npm 11 ships by default                                              | **None** -- project uses Bun as package manager                                                                | No              |

### Summary

**This codebase has zero Node.js API breaking changes to address.** The action uses a narrow API surface: `child_process.spawnSync`, `fs.existsSync`, `process.env`, and the `@actions/*` toolkit. None of these APIs have breaking changes between Node.js 20 and 24.

## GitHub Actions node24 Runtime

### Timeline (HIGH confidence)

| Date           | Event                                                             |
| -------------- | ----------------------------------------------------------------- |
| September 2025 | Runner v2.328.0 ships with node24 support; node20 remains default |
| June 2, 2026   | Runners switch to node24 as default                               |
| Fall 2026      | node20 fully removed from runners                                 |

### action.yml Change

The only required change is in `action.yml`:

```yaml
# Before
runs:
  using: 'node20'
  main: 'dist/nx-set-shas.js'

# After
runs:
  using: 'node24'
  main: 'dist/nx-set-shas.js'
```

### Testing Before June 2026

Set `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` as a workflow env variable to force the runner to use node24 for all actions. This is the recommended way to validate before the switch date.

```yaml
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
```

## Dependency Compatibility

### @actions/core (currently 1.11.1)

| Version          | Module System | Node.js Compat | Notes                                                 |
| ---------------- | ------------- | -------------- | ----------------------------------------------------- |
| 1.11.1 (current) | CJS           | node20+        | Works on node24 -- pure JS, no native deps            |
| 2.0.x            | CJS           | node20+        | Adds `@actions/exec` dep, no API breaks for our usage |
| 3.0.0 (latest)   | **ESM-only**  | node24+        | `"type": "module"`, exports only `import`             |

**Recommendation: Stay on @actions/core ^1.11.1.** (MEDIUM confidence)

Rationale:

- The 1.x line is pure CJS JavaScript with no native dependencies. It will run on any Node.js version.
- Bun bundles everything into a single `dist/nx-set-shas.js` file, so module system (CJS vs ESM) of dependencies is irrelevant at runtime.
- The APIs used (`getInput`, `setOutput`, `setFailed`, `getBooleanInput`, `exportVariable`) are stable across all versions.
- Upgrading to 3.0.0 (ESM-only) would require verifying Bun's ESM bundling path and testing the deep import `@actions/github/lib/utils` -- unnecessary risk for zero benefit.
- The PROJECT.md constraint says "Update deps only when they block Node.js 24 compatibility." They do not block it.

### @actions/github (currently 6.0.1)

| Version         | Module System | Octokit           | Notes                               |
| --------------- | ------------- | ----------------- | ----------------------------------- |
| 6.0.1 (current) | CJS           | @octokit/core 5.x | Works on node24 -- pure JS          |
| 9.0.0 (latest)  | **ESM-only**  | @octokit/core 7.x | `"type": "module"`, uses undici 6.x |

**Recommendation: Stay on @actions/github ^6.0.1.** (MEDIUM confidence)

Rationale:

- Same as above: pure CJS JS, bundled by Bun, runs on any Node.js version.
- The codebase uses `github.context`, `github.getOctokit()`, and the deep import `@actions/github/lib/utils` for the `GitHub` type. The 9.0.0 ESM-only version restructures exports and this deep import would break.
- No functional benefit to upgrading for this migration.

### @types/node

| Current | Target                | Notes                                    |
| ------- | --------------------- | ---------------------------------------- |
| 20.19.9 | 24.12.0 (latest 24.x) | Type definitions only; no runtime impact |

**Recommendation: Update @types/node to ^24.12.0.** (HIGH confidence)

This is a dev dependency that only affects type checking. Updating ensures TypeScript recognizes Node.js 24 APIs and flags any deprecated API usage at compile time. This is the one dependency that should definitely be updated.

### TypeScript (currently 5.8.3)

**Recommendation: Keep as-is.** (HIGH confidence)

TypeScript 5.8.3 supports `ES2023` target and `nodenext` module resolution. No changes needed for Node.js 24 compatibility.

### tsconfig.json

**Recommendation: Consider updating `target` and `lib` to `ES2024`.** (LOW confidence)

Node.js 24 (V8 13.6) supports ES2024 features including `Promise.withResolvers`, `ArrayBuffer.resize`, and `Object.groupBy`. However, since the codebase does not use these features and the scope is minimal migration, keeping `ES2023` is fine. Flag for optional improvement.

### Bun (currently 1.2.19)

**Recommendation: Keep as-is.** (HIGH confidence)

Bun is the build tool, not the runtime. The output `dist/nx-set-shas.js` runs on the GitHub Actions Node.js runtime. Bun's `--target node` flag produces Node.js-compatible output regardless of the Node.js version.

### Volta Configuration

| Current             | Target                                |
| ------------------- | ------------------------------------- |
| `"node": "20.19.4"` | `"node": "24.12.0"` (latest 24.x LTS) |

**Recommendation: Update Volta pin to Node.js 24 LTS.** (HIGH confidence)

Node.js 24.11.0 "Krypton" became LTS on October 28, 2025. The latest LTS is 24.12.0 (December 2025). Update the Volta pin and `engines` field.

### package.json engines

| Current          | Target           |
| ---------------- | ---------------- |
| `"node": ">=20"` | `"node": ">=24"` |

**Recommendation: Update to `>=24`.** (HIGH confidence)

This communicates to contributors that Node.js 24 is the minimum. Since the action runs on node24 runtime only, there is no reason to support older versions.

## What NOT to Change (and Why)

| Item                           | Why Keep As-Is                                                          |
| ------------------------------ | ----------------------------------------------------------------------- |
| `@actions/core` version        | Pure CJS JS, works on node24, no API changes needed                     |
| `@actions/github` version      | Pure CJS JS, works on node24, deep import would break on v9             |
| Bun version/config             | Build tool, not runtime; output is Node.js-compatible                   |
| TypeScript version             | 5.8.3 fully supports the target config                                  |
| Build command                  | `bun build ./nx-set-shas.ts --outdir ./dist --target node` -- unchanged |
| Source code (`nx-set-shas.ts`) | No Node.js API breaking changes affect this code                        |
| Prettier/Husky/lint-staged     | Dev tooling, unrelated to Node.js runtime                               |

## Recommended Changes Summary

| Change                                                    | File(s)            | Risk                 | Required    |
| --------------------------------------------------------- | ------------------ | -------------------- | ----------- |
| `using: 'node20'` -> `using: 'node24'`                    | action.yml         | None                 | Yes         |
| `@types/node` 20.x -> 24.x                                | package.json       | None (types only)    | Yes         |
| Volta pin 20.19.4 -> 24.12.0                              | package.json       | None                 | Yes         |
| engines `>=20` -> `>=24`                                  | package.json       | None                 | Yes         |
| Version 4.4.0 -> 5.0.0                                    | package.json       | None (semver signal) | Yes         |
| tsconfig target/lib ES2023 -> ES2024                      | tsconfig.json      | Very low             | Optional    |
| Add CI workflow with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` | .github/workflows/ | None                 | Recommended |

## Confidence Levels

| Finding                                     | Confidence | Basis                                                                                            |
| ------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------ |
| node24 runtime timeline                     | HIGH       | GitHub official changelog (Sep 2025)                                                             |
| action.yml `using: 'node24'` syntax         | HIGH       | GitHub Actions docs, actions/setup-node already uses it                                          |
| @actions/core 1.11.1 works on node24        | MEDIUM     | Pure CJS JS with no native deps; no official compat matrix found, but zero reason it would break |
| @actions/github 6.0.1 works on node24       | MEDIUM     | Same rationale as above                                                                          |
| No Node.js API breaking changes in codebase | HIGH       | Verified against official v20->v22 and v22->v24 migration guides                                 |
| @types/node 24.x available                  | HIGH       | Verified on npm registry (24.12.0 exists)                                                        |
| Node.js 24 LTS version 24.12.0              | HIGH       | nodejs.org release blog                                                                          |
| Volta supports Node.js 24                   | HIGH       | Volta 2.0.2 supports any Node.js version                                                         |

## Sources

- [GitHub Changelog: Deprecation of Node 20 on GitHub Actions runners](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/)
- [Node.js Official: v20 to v22 Migration Guide](https://nodejs.org/en/blog/migrations/v20-to-v22)
- [Node.js Official: v22 to v24 Migration Guide](https://nodejs.org/en/blog/migrations/v22-to-v24)
- [Node.js Official: v24.0.0 Release](https://nodejs.org/en/blog/release/v24.0.0)
- [Node.js 24 Becomes LTS (NodeSource)](https://nodesource.com/blog/nodejs-24-becomes-lts)
- [actions/toolkit GitHub repository](https://github.com/actions/toolkit)
- [npm registry: @actions/core](https://www.npmjs.com/package/@actions/core)
- [npm registry: @actions/github](https://www.npmjs.com/package/@actions/github)
- [OpenJS Foundation: Node.js 24 Released](https://openjsf.org/blog/nodejs-24-released)
