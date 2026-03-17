# Phase 3: Source Fixes and Build - Research

**Researched:** 2026-03-17
**Domain:** Node.js 24 API compatibility audit, @actions/\* ESM upgrades, Bun bundler rebuild
**Confidence:** HIGH

## Summary

Phase 3 covers three distinct activities: (1) auditing all Node.js built-in API usage in source files for Node.js 24 compatibility, (2) upgrading `@actions/core` from 1.11.1 to 3.0.0 and `@actions/github` from 6.0.1 to 9.0.0, and (3) rebuilding `dist/nx-set-shas.js` with Bun and verifying CI passes.

Research confirms the audit will find zero breaking Node.js API changes -- the codebase uses only `spawnSync`, `execSync`, `existsSync`, `process.chdir`, `process.stdout.write`, and `process.env`, none of which have breaking changes between Node.js 20 and 24. The `@actions/*` upgrades are well-defined: both packages are now ESM-only (`"type": "module"`) with explicit `exports` maps. The project already uses `"type": "module"` and Bun's bundler defaults to ESM output, so the upgrade path is straightforward. The only current `tsc` error (TS2307 on `@actions/github/lib/utils`) is resolved by `@actions/github@9.0.0` which explicitly exports `"./lib/utils"` in its exports map.

**Primary recommendation:** Execute the three-commit sequence defined in CONTEXT.md: audit doc first, then dependency upgrade, then rebuild and CI verification. The `catch (e)` at line 78 is NOT a compilation issue (strict mode is disabled in tsconfig), but verify after the upgrade since @actions/core 3.0.0 types may surface new issues.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions

- TS2307 resolution: upgrading `@actions/github` to 9.0.0 fixes the `@actions/github/lib/utils` import -- no source code changes needed for the type import
- Bump `@actions/core` from ^1.11.1 to ^3.0.0 (ESM-only, adds Node 24 support)
- Bump `@actions/github` from ^6.0.1 to ^9.0.0 (ESM-only, fixes TypeScript compilation, updated octokit)
- Audit scope: Node.js built-in APIs only (spawnSync, execSync, existsSync, process._); @actions/_ validated by Phase 1
- Audit format: API-by-API checklist table in `03-AUDIT.md`
- Note pre-commit hook `npm run build` vs `bun run build` discrepancy as a finding, do not fix
- Build validation sequence: `bun run build` -> `tsc --noEmit` -> review `git diff dist/` -> commit
- Push to remote and verify CI green within Phase 3
- Commit ordering: (1) audit doc, (2) dependency bump, (3) rebuild dist/
- Out of scope: `/dev/null` git argument, `stripNewLineEndings()` latent bug (SRCF-01), stdio normalization (SRCF-02), pre-commit hook npm/bun discrepancy fix

### Claude's Discretion

- Exact audit table format and depth of deprecation research
- How to verify CI results (polling, waiting, etc.)
- Fix approach for any unexpected tsc errors after the upgrade
- Whether to combine any commits if they are trivially small

### Deferred Ideas (OUT OF SCOPE)

None -- discussion stayed within phase scope.

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID      | Description                                                                       | Research Support                                                                                                                              |
| ------- | --------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| AUDT-01 | Audit all Node.js API usage in `nx-set-shas.ts` for Node.js 24 compatibility      | Node.js v22-to-v24 migration guide confirms zero breaking changes for spawnSync, existsSync, process.chdir, process.stdout.write, process.env |
| AUDT-02 | Audit all Node.js API usage in `tools/pre-commit.ts` for Node.js 24 compatibility | Same -- execSync and process.exit have no breaking changes in Node.js 24                                                                      |
| AUDT-03 | Document audit findings (APIs checked, changes needed, no-change confirmations)   | Audit table format defined in CONTEXT.md; research provides Node.js 24 changelog data to populate it                                          |
| BVAL-01 | Rebuild `dist/nx-set-shas.js` with Bun targeting Node.js 24                       | Bun bundler handles ESM-only deps natively; existing `bun build` command works unchanged                                                      |

</phase_requirements>

## Standard Stack

### Core

| Library           | Version                 | Purpose                                                                  | Why Standard                                                               |
| ----------------- | ----------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| `@actions/core`   | 3.0.0                   | GitHub Actions I/O (getInput, setOutput, setFailed, exportVariable)      | Official toolkit, ESM-only, explicit Node 24 support added in 2.0.0        |
| `@actions/github` | 9.0.0                   | GitHub API client (getOctokit, context) + GitHub type from `./lib/utils` | Official toolkit, ESM-only, exports `./lib/utils` subpath, updated octokit |
| Bun               | 1.2.19 (packageManager) | Bundler for `dist/nx-set-shas.js`                                        | Already project standard; handles ESM deps natively                        |
| TypeScript        | ^5.8.3                  | Type checking via `tsc --noEmit`                                         | Already project standard                                                   |

### Supporting

| Library       | Version  | Purpose                  | When to Use                  |
| ------------- | -------- | ------------------------ | ---------------------------- |
| `@types/node` | ^24.12.0 | Node.js type definitions | Already installed in Phase 2 |

### Alternatives Considered

None -- all decisions are locked by CONTEXT.md.

**Installation:**

```bash
bun add @actions/core@^3.0.0 @actions/github@^9.0.0
```

## Architecture Patterns

### Dependency Upgrade Pattern

The `@actions/*` upgrade is a semver-major bump but the API surface used by this project is fully preserved:

**@actions/core 1.11.1 -> 3.0.0 breaking changes:**

- 2.0.0: Add Node 24 support, bump `@actions/http-client` to 3.0.0
- 3.0.0: Package is ESM-only (CJS consumers must use dynamic `import()`)

**@actions/github 6.0.1 -> 9.0.0 breaking changes:**

- 7.0.0: Bump `@actions/http-client` to 3.0.1
- 8.0.0: Update octokit deps (core ^7, plugin-paginate-rest ^14, plugin-rest-endpoint-methods ^17, request ^10, request-error ^7), minimum Node.js 20
- 9.0.0: ESM-only, fixes TypeScript compilation by migrating to ESM

**APIs used by this project (all confirmed present in 3.0.0/9.0.0):**

- `core.getInput()` -- confirmed in @actions/core 3.0.0
- `core.getBooleanInput()` -- confirmed
- `core.setOutput()` -- confirmed
- `core.setFailed()` -- confirmed
- `core.exportVariable()` -- confirmed
- `github.getOctokit()` -- confirmed in @actions/github 9.0.0
- `github.context` -- confirmed
- `octokit.request()` -- confirmed (uses updated @octokit/request ^10)
- `GitHub` type from `@actions/github/lib/utils` -- confirmed via exports map

### ESM Compatibility

The project already has `"type": "module"` in package.json. The ESM-only nature of the new packages aligns perfectly. Bun's bundler outputs ESM by default and resolves ESM imports natively. No module system changes needed.

### Build Flow

```
Source (nx-set-shas.ts)
  |-- bun build ./nx-set-shas.ts --outdir ./dist --target node
  |-- Output: dist/nx-set-shas.js (single bundled file)
  |-- tsc --noEmit (type checking only, separate from build)
```

### Anti-Patterns to Avoid

- **Do NOT add `--format=cjs`:** The project is ESM, the deps are ESM, Bun defaults to ESM. Adding CJS format would break things.
- **Do NOT modify the import at line 3:** `import { GitHub } from '@actions/github/lib/utils'` is a valid import with @actions/github 9.0.0 -- the exports map explicitly supports this subpath.
- **Do NOT enable strict mode in tsconfig:** This would surface the `catch (e)` typing issue and is out of scope for the migration.

## Don't Hand-Roll

| Problem              | Don't Build                      | Use Instead                                             | Why                                                      |
| -------------------- | -------------------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| Node.js 24 API audit | Manual reading of Node.js source | Official v22-to-v24 migration guide + deprecations docs | Authoritative, maintained by Node.js team                |
| ESM interop          | Custom require/import wrappers   | Bun bundler native ESM handling                         | Bun resolves ESM/CJS interop at bundle time              |
| CI verification      | Manual test reproduction locally | Push to remote, `gh run watch`                          | CI runs the action via `uses: ./` which is the real test |

## Common Pitfalls

### Pitfall 1: Octokit API Response Shape Changes

**What goes wrong:** @actions/github 9.0.0 uses updated @octokit/core ^7 and @octokit/request ^10. Response types or error shapes may differ from v6.
**Why it happens:** Major octokit version bumps can change response typing or error handling.
**How to avoid:** The project uses raw `octokit.request()` with string URL templates, not typed endpoint methods. This is resilient to octokit type changes. The actual runtime responses come from GitHub's API which is versioned separately. Verify `tsc --noEmit` passes after the upgrade.
**Warning signs:** TypeScript errors on `octokit.request()` calls or response destructuring.

### Pitfall 2: Bun Lockfile Conflicts After Major Dependency Bump

**What goes wrong:** `bun add` updates `bun.lock` but transitive dependency resolution may change significantly with major version bumps.
**Why it happens:** @actions/github 9.0.0 pulls in entirely different octokit versions than 6.0.1.
**How to avoid:** Run `bun install` after the upgrade, verify `bun.lock` is updated, check that `bun run build` succeeds.
**Warning signs:** Build errors about missing modules or version conflicts.

### Pitfall 3: Pre-commit Hook Runs During Commit

**What goes wrong:** The pre-commit hook runs `npm run build` (which delegates to `bun build`) and auto-stages dist/ changes. If the upgrade changes the dist/ output significantly, the hook may interfere with the planned commit sequence.
**Why it happens:** The hook is designed to keep dist/ in sync, but during a major dependency upgrade, the dist/ changes are intentional and should be in a specific commit.
**How to avoid:** Be aware the hook will fire. The planned commit sequence (audit doc first, then deps, then dist/) means the dist/ rebuild commit will naturally include the hook's auto-staged changes.
**Warning signs:** Unexpected files staged after commit.

### Pitfall 4: CI Fails Due to Node.js Version Mismatch

**What goes wrong:** @actions/core 3.0.0 and @actions/github 9.0.0 require Node.js 20+ at minimum. The action.yml declares `node24`. If the runner hasn't picked up Node 24 yet, the action may fail.
**Why it happens:** GitHub Actions runners are in transition from Node 20 to Node 24.
**How to avoid:** The `action.yml` already declares `node24` (set in Phase 2). The fork CI uses `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` if needed. Phase 1 already verified this works.
**Warning signs:** CI errors about unsupported Node.js version.

## Code Examples

### Node.js APIs Used in nx-set-shas.ts (Audit Reference)

```typescript
// Line 4: child_process.spawnSync -- no breaking changes in Node.js 24
import { spawnSync } from 'node:child_process';

// Line 5: fs.existsSync -- no breaking changes in Node.js 24
// Note: Node.js 24 adds runtime warning for invalid inputs to existsSync(),
// but this codebase passes valid string paths only
import { existsSync } from 'node:fs';

// Lines 40, 53, 63, 98, 103: spawnSync('git', [...], { encoding: 'utf-8' })
// Stable API, no changes needed

// Line 32: process.chdir(workingDirectory) -- stable
// Lines 33-34, 87-89, etc.: process.stdout.write() -- stable
// Line 12: process.env.GITHUB_TOKEN = ... -- stable
```

### Node.js APIs Used in tools/pre-commit.ts (Audit Reference)

```typescript
// Line 1: child_process.execSync -- no breaking changes in Node.js 24
import { execSync } from 'node:child_process';

// Lines 13, 18, 26, 34: execSync(cmd, { stdio: ['pipe', 'pipe', 'pipe'], encoding: 'utf-8' })
// Stable API, no changes needed

// Lines 8, 41: process.exit() -- stable
```

### Dependency Upgrade Command

```bash
# Upgrade both @actions/* packages
bun add @actions/core@^3.0.0 @actions/github@^9.0.0
```

### Verification Sequence

```bash
# 1. Type check (should now pass with @actions/github 9.0.0 fixing TS2307)
npx tsc --noEmit

# 2. Build
bun run build

# 3. Review dist changes
git diff --stat dist/

# 4. Push and verify CI
git push
gh run watch
```

## State of the Art

| Old Approach              | Current Approach                           | When Changed | Impact                                                             |
| ------------------------- | ------------------------------------------ | ------------ | ------------------------------------------------------------------ |
| @actions/core 1.x (CJS)   | @actions/core 3.0.0 (ESM-only)             | 2025         | Must use ESM imports; project already ESM                          |
| @actions/github 6.x (CJS) | @actions/github 9.0.0 (ESM-only)           | 2025         | ESM-only; fixes TS compilation; exports ./lib/utils                |
| Node.js 20 runtime        | Node.js 24 runtime                         | May 2025     | No breaking changes for this codebase's API surface                |
| Octokit 5.x/6.x bundled   | Octokit 7.x bundled in @actions/github 9.x | 2025         | Uses @octokit/core ^7, request ^10; raw request() calls still work |

**Deprecated/outdated:**

- `@actions/core` 1.x: No official Node 24 support (though it works in practice per Phase 1 testing)
- `@actions/github` 6.x: Has TS2307 error with `@types/node` 24.x due to missing exports map for `./lib/utils`

## Open Questions

1. **Will @octokit/request ^10 change response shapes?**
   - What we know: The project uses raw `octokit.request()` with string templates and destructures `{ data: { workflow_id } }` and `{ data: { workflow_runs } }`. These are GitHub REST API responses, not octokit-typed shapes.
   - What's unclear: Whether @octokit/request ^10 changes the response wrapper or adds stricter typing
   - Recommendation: Run `tsc --noEmit` after upgrade; if it passes, the types are compatible. Runtime behavior depends on GitHub API, not octokit version.

2. **Will Bun produce a significantly different dist/ output with ESM-only deps?**
   - What we know: Current dist/ bundles CJS @actions/\* deps into a single file. ESM deps will be resolved differently.
   - What's unclear: Whether the output size or structure changes dramatically
   - Recommendation: Review `git diff dist/` after build; large changes are expected and acceptable since the deps are fundamentally different.

## Validation Architecture

### Test Framework

| Property           | Value                                                |
| ------------------ | ---------------------------------------------------- |
| Framework          | GitHub Actions CI (integration tests via `uses: ./`) |
| Config file        | `.github/workflows/test.yml`                         |
| Quick run command  | `npx tsc --noEmit && bun run build`                  |
| Full suite command | `git push` + CI green on test.yml (3 OS matrix)      |

### Phase Requirements to Test Map

| Req ID  | Behavior                             | Test Type   | Automated Command                   | File Exists?      |
| ------- | ------------------------------------ | ----------- | ----------------------------------- | ----------------- |
| AUDT-01 | Node.js API audit for nx-set-shas.ts | manual-only | N/A -- documentation deliverable    | N/A               |
| AUDT-02 | Node.js API audit for pre-commit.ts  | manual-only | N/A -- documentation deliverable    | N/A               |
| AUDT-03 | Document audit findings              | manual-only | N/A -- documentation deliverable    | N/A               |
| BVAL-01 | Rebuild dist/nx-set-shas.js          | smoke       | `bun run build && npx tsc --noEmit` | N/A (CI workflow) |

Note: AUDT-01/02/03 are documentation requirements, not code changes. Validation is review of the audit document. BVAL-01 is validated by successful build + CI green. There are no unit tests in this project -- all testing is integration-level via CI workflows that run the action with `uses: ./`.

### Sampling Rate

- **Per task commit:** `npx tsc --noEmit && bun run build`
- **Per wave merge:** N/A (single branch, push-based)
- **Phase gate:** CI green on all 3 OS matrix runners (ubuntu, macos, windows)

### Wave 0 Gaps

None -- existing CI infrastructure (test.yml with 3-OS matrix, integration-test-workflow.yml) covers all phase requirements. No unit test framework needed since the project uses integration testing only.

## Sources

### Primary (HIGH confidence)

- npm registry `@actions/core@3.0.0` -- verified `type: "module"`, exports map, version
- npm registry `@actions/github@9.0.0` -- verified `type: "module"`, exports map with `"./lib/utils"`, version
- [actions/toolkit RELEASES.md (core)](https://github.com/actions/toolkit/blob/main/packages/core/RELEASES.md) -- full changelog 1.0.0 through 3.0.0
- [actions/toolkit RELEASES.md (github)](https://github.com/actions/toolkit/blob/main/packages/github/RELEASES.md) -- full changelog 1.0.0 through 9.0.0
- [Node.js v22 to v24 migration guide](https://nodejs.org/en/blog/migrations/v22-to-v24) -- breaking changes list
- [jsDocs.io @actions/core 3.0.0](https://www.jsdocs.io/package/@actions/core) -- confirmed all 5 APIs exist
- Local `tsc --noEmit` run -- confirmed TS2307 is the only current error
- Local `tsconfig.json` read -- confirmed strict mode is NOT enabled (catch variables typed as `any`)

### Secondary (MEDIUM confidence)

- [Bun bundler docs](https://bun.com/docs/bundler) -- ESM handling and target node behavior
- [Node.js deprecated APIs docs](https://nodejs.org/api/deprecations.html) -- deprecation list

### Tertiary (LOW confidence)

None -- all findings verified with primary sources.

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH - npm registry and RELEASES.md provide authoritative version/changelog data
- Architecture: HIGH - project structure is simple, ESM alignment is clean, verified locally
- Pitfalls: HIGH - based on actual dependency changelogs and local tsc verification
- Node.js 24 audit: HIGH - official migration guide confirms no breaking changes for used APIs

**Research date:** 2026-03-17
**Valid until:** 2026-04-17 (stable -- packages are released, Node.js 24 is LTS)
