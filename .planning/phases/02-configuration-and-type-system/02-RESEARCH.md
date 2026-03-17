# Phase 2: Configuration and Type System - Research

**Researched:** 2026-03-17
**Domain:** GitHub Actions runtime declaration, Node.js toolchain configuration, TypeScript compiler settings
**Confidence:** HIGH

## Summary

Phase 2 is a straightforward configuration update across four files (`action.yml`, `package.json`, `tsconfig.json`, and `@types/node` dependency) plus cleanup of the temporary `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` flag from five CI workflow files. All changes are well-documented with authoritative sources and the codebase is small enough that the impact surface is fully visible.

The TypeScript target bump from ES2023 to ES2024 is a one-step change per the official Microsoft TypeScript Node Target Mapping. The `@types/node` bump from 20.x to 24.x is the highest-risk item, but Phase 1 already confirmed zero runtime issues on Node.js 24, so any type errors would be purely at the type-checking level. There is one pre-existing `tsc --noEmit` error (`TS2307: Cannot find module '@actions/github/lib/utils'`) that exists today and is unrelated to this migration.

**Primary recommendation:** Apply all configuration changes in sequence (action.yml, package.json, tsconfig.json, @types/node), then run `tsc --noEmit` to surface any new errors. Remove the FORCE flag from all 5 workflow files in the same phase. The FORCE flag removal is safe because once `action.yml` declares `node24`, GitHub Actions runners will use Node.js 24 natively.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions

- Set both `target` and `lib` to `ES2024`, per the official Microsoft TypeScript Node Target Mapping for Node.js 24
- Current values are `ES2023` (matching Node.js 20) -- this is a one-step bump
- `module` stays `nodenext` (unchanged)
- Volta pin: set to latest Node.js 24.x LTS version available at plan time (researcher resolves exact version)
- `engines.node`: bump from `>=20` to `>=24`
- This is a breaking change (contributors must have Node.js 24) -- acceptable since the entire migration is a major version bump
- Remove `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` from all CI workflows in this phase
- The flag was added in Phase 1 (commit `1cc440b`) as `[TEMP]` and is redundant once `action.yml` declares `node24`
- Bump `@types/node` from `^20.x` to `^24.x`
- Run `tsc --noEmit` after the bump
- Fix any compilation errors inline in Phase 2 -- the build must be green after config changes

### Claude's Discretion

- Exact Node.js 24.x version for Volta pin (resolve latest LTS at research time)
- Commit granularity within the phase (single commit vs separate config/types commits)
- Order of config file changes
- How to verify tsc passes after changes

### Deferred Ideas (OUT OF SCOPE)

None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID      | Description                                                          | Research Support                                                                                           |
| ------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| RUNT-01 | Update `action.yml` runtime declaration from `node20` to `node24`    | GitHub Actions officially supports `using: 'node24'` since runner v2.327.0; confirmed via GitHub Changelog |
| RUNT-02 | Update Volta Node.js pin in `package.json` to Node.js 24.x           | Latest LTS is 24.14.0 (released 2026-03-11); use this as the Volta pin                                     |
| RUNT-03 | Update `engines.node` in `package.json` to require Node.js >= 24     | Straightforward semver field change from `>=20` to `>=24`                                                  |
| TSCO-01 | Update `tsconfig.json` `target` per TypeScript Node Target Mapping   | Official mapping: Node.js 24 -> ES2024 target (verified via Microsoft wiki)                                |
| TSCO-02 | Update `tsconfig.json` `lib` per TypeScript Node Target Mapping      | Official mapping: Node.js 24 -> ES2024 lib (verified via Microsoft wiki)                                   |
| TYPE-01 | Update `@types/node` from 20.x to 24.x                               | Latest available: `@types/node@24.12.0` on npm; use `^24.12.0`                                             |
| TYPE-02 | Fix all TypeScript compilation errors surfaced by `@types/node` 24.x | Pre-existing TS2307 error unrelated to migration; no new errors expected based on Phase 1 findings         |

</phase_requirements>

## Standard Stack

### Core

| Library     | Version  | Purpose                           | Why Standard                                    |
| ----------- | -------- | --------------------------------- | ----------------------------------------------- |
| TypeScript  | ^5.8.3   | Type checking (already installed) | Project uses `tsc --noEmit` for type validation |
| @types/node | ^24.12.0 | Node.js type definitions          | Matches Node.js 24 runtime; latest 24.x on npm  |
| Node.js     | 24.14.0  | Runtime (Volta pin)               | Latest LTS as of 2026-03-11                     |

### Supporting

No new libraries needed. All changes are configuration-only.

### Alternatives Considered

None -- all choices are locked decisions from CONTEXT.md.

## Architecture Patterns

### Configuration Files to Modify

```
action.yml                              # line 45: using: 'node20' -> 'node24'
package.json                            # engines.node, volta.node, @types/node version
tsconfig.json                           # target and lib: ES2023 -> ES2024
.github/workflows/test.yml             # remove FORCE flag
.github/workflows/test-integration.yml # remove FORCE flag
.github/workflows/format.yml           # remove FORCE flag
.github/workflows/publish.yml          # remove FORCE flag
.github/workflows/integration-test-workflow.yml  # remove FORCE flag
```

### Pattern: action.yml Runtime Declaration

**What:** GitHub Actions `runs.using` field specifies the Node.js runtime version.
**Current:** `using: 'node20'`
**Target:** `using: 'node24'`

```yaml
# Source: https://docs.github.com/en/actions/reference/workflows-and-actions/metadata-syntax
runs:
  using: 'node24'
  main: 'dist/nx-set-shas.js'
```

**Note:** GitHub skipped `node22` entirely. The valid values are `node12`, `node16`, `node20`, and `node24`. Runner v2.327.0+ is required.

### Pattern: Volta Pin

**What:** Volta reads `volta.node` from `package.json` to auto-switch Node.js version.
**Current:** `"node": "20.19.4"`
**Target:** `"node": "24.14.0"`

```json
{
  "volta": {
    "node": "24.14.0"
  }
}
```

### Pattern: FORCE Flag Cleanup

**What:** Remove the `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true` environment variable from all 5 workflow files.
**Why safe:** Once `action.yml` declares `node24`, the runner uses Node.js 24 natively. The FORCE flag is only needed when `action.yml` still says `node20` but you want to test on 24.

Each workflow has the same structure:

```yaml
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
```

**Important:** There are **5** workflow files with the flag (not 3 as CONTEXT.md mentions):

1. `.github/workflows/test.yml`
2. `.github/workflows/test-integration.yml`
3. `.github/workflows/format.yml`
4. `.github/workflows/publish.yml`
5. `.github/workflows/integration-test-workflow.yml`

If the `env:` block contains only this one variable, remove the entire `env:` block. If there are other variables, remove only this line.

### Anti-Patterns to Avoid

- **Do NOT set `module` to anything other than `nodenext`:** The CONTEXT.md explicitly locks `module` as unchanged. The TypeScript wiki notes a potential switch to `node20` after TS 5.9, but that is deferred (TSCO-03 is v2).
- **Do NOT bump `@types/node` to 25.x:** The project targets Node.js 24 specifically; use 24.x types.
- **Do NOT fix the pre-existing TS2307 error:** The `@actions/github/lib/utils` import error exists today on the current branch and is not caused by this migration. It is out of scope.

## Don't Hand-Roll

| Problem                   | Don't Build                           | Use Instead                    | Why                                            |
| ------------------------- | ------------------------------------- | ------------------------------ | ---------------------------------------------- |
| Node.js version detection | Custom version-check scripts          | Volta pin + `engines` field    | Volta auto-switches; npm/bun warns on mismatch |
| Runtime declaration       | Manual runner configuration           | `action.yml` `using: 'node24'` | GitHub Actions native mechanism                |
| Type compatibility        | Manual type annotations for Node APIs | `@types/node@24`               | DefinitelyTyped maintains accurate types       |

## Common Pitfalls

### Pitfall 1: Missing Workflow Files for FORCE Flag Removal

**What goes wrong:** CONTEXT.md mentions 3 workflows but there are actually 5 with the FORCE flag.
**Why it happens:** The context was written from memory; Phase 1 summary correctly states "all 5 workflow files."
**How to avoid:** Search for `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` across all `.github/workflows/*.yml` files and remove from every match.
**Warning signs:** CI still shows `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` in workflow logs after the change.

### Pitfall 2: Pre-existing TS2307 Error Confusion

**What goes wrong:** `tsc --noEmit` reports `TS2307: Cannot find module '@actions/github/lib/utils'` and developer thinks the migration broke something.
**Why it happens:** This error exists today with the current `@types/node@20` setup. It is caused by `@actions/github` not shipping type declarations for its deep `lib/utils` import path.
**How to avoid:** Document this as a known pre-existing error. The success criterion for TYPE-02 is "no NEW errors introduced by the migration." This specific error is not new.
**Warning signs:** Attempting to fix this error would pull in out-of-scope dependency work.

### Pitfall 3: Volta Pin Version Drift

**What goes wrong:** Volta pin set to a version that doesn't exist yet or is too old.
**Why it happens:** Node.js releases new patch versions frequently.
**How to avoid:** Use `24.14.0` which is the latest LTS as of 2026-03-11. This is verified via the official Node.js releases page.

### Pitfall 4: Empty `env:` Block After Flag Removal

**What goes wrong:** Removing the FORCE flag line but leaving an empty `env:` key in the YAML.
**Why it happens:** Mechanical editing without checking if the env block has other variables.
**How to avoid:** Check each workflow: if `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` is the only env variable, remove the entire `env:` block (both the key and value).

## Code Examples

### action.yml Change

```yaml
# Before (line 45)
runs:
  using: 'node20'
  main: 'dist/nx-set-shas.js'

# After
runs:
  using: 'node24'
  main: 'dist/nx-set-shas.js'
```

### package.json Changes

```json
{
  "engines": {
    "node": ">=24"
  },
  "volta": {
    "node": "24.14.0"
  },
  "devDependencies": {
    "@types/node": "^24.12.0"
  }
}
```

### tsconfig.json Change

```json
{
  "compilerOptions": {
    "lib": ["ES2024"],
    "module": "nodenext",
    "target": "ES2024"
  }
}
```

### Workflow FORCE Flag Removal

```yaml
# Before (each of 5 workflow files)
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
# After: remove the entire env block (if no other env vars exist)
```

## State of the Art

| Old Approach                                  | Current Approach                     | When Changed                          | Impact                                                                |
| --------------------------------------------- | ------------------------------------ | ------------------------------------- | --------------------------------------------------------------------- |
| `using: 'node20'` in action.yml               | `using: 'node24'`                    | Runner v2.327.0 (2025)                | Native Node.js 24 support                                             |
| `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` env flag | Not needed with `node24` declaration | Same runner version                   | Flag was always a testing bridge                                      |
| `target: "ES2023"` for Node 20                | `target: "ES2024"` for Node 24       | TypeScript Node Target Mapping update | Unlocks ES2024 features (Object.groupBy, Promise.withResolvers, etc.) |
| `@types/node@20`                              | `@types/node@24`                     | DefinitelyTyped tracking Node 24      | Type definitions match runtime APIs                                   |

**GitHub Actions Node.js timeline:**

- Node 20 deprecation starts: runners switch to Node 24 by default on June 2, 2026
- Node 20 removal: fall 2026
- Action using `node24` works today on runner v2.327.0+

## Open Questions

1. **Pre-existing TS2307 error**
   - What we know: `@actions/github/lib/utils` deep import has no type declarations
   - What's unclear: Whether this error existed before the fork or was introduced by a dependency version
   - Recommendation: Ignore in this phase; it is pre-existing and unrelated to Node.js 24 migration

2. **Bun lockfile update after @types/node bump**
   - What we know: The project uses `bun` as package manager (`packageManager: "bun@1.2.19"`)
   - What's unclear: Whether `bun install` needs to be run to update the lockfile after changing `@types/node` version
   - Recommendation: Run `bun install` after modifying `package.json` to update `bun.lock`

## Validation Architecture

### Test Framework

| Property           | Value                                                |
| ------------------ | ---------------------------------------------------- |
| Framework          | GitHub Actions CI workflows (no unit test framework) |
| Config file        | `.github/workflows/test.yml`, `test-integration.yml` |
| Quick run command  | `npx tsc --noEmit` (type checking only)              |
| Full suite command | Push to branch, verify all 5 CI workflows pass       |

### Phase Requirements -> Test Map

| Req ID  | Behavior                       | Test Type | Automated Command                                                                                                 | File Exists?       |
| ------- | ------------------------------ | --------- | ----------------------------------------------------------------------------------------------------------------- | ------------------ |
| RUNT-01 | action.yml declares node24     | smoke     | `git grep "using: 'node24'" -- action.yml`                                                                        | N/A (grep check)   |
| RUNT-02 | Volta pin is 24.14.0           | smoke     | `node -e "const p=require('./package.json'); console.assert(p.volta.node==='24.14.0')"`                           | N/A (inline check) |
| RUNT-03 | engines.node requires >=24     | smoke     | `node -e "const p=require('./package.json'); console.assert(p.engines.node==='>=24')"`                            | N/A (inline check) |
| TSCO-01 | tsconfig target is ES2024      | smoke     | `node -e "const t=require('./tsconfig.json'); console.assert(t.compilerOptions.target==='ES2024')"`               | N/A (inline check) |
| TSCO-02 | tsconfig lib includes ES2024   | smoke     | `node -e "const t=require('./tsconfig.json'); console.assert(t.compilerOptions.lib[0]==='ES2024')"`               | N/A (inline check) |
| TYPE-01 | @types/node is 24.x            | smoke     | `node -e "const p=require('./package.json'); console.assert(p.devDependencies['@types/node'].startsWith('^24'))"` | N/A (inline check) |
| TYPE-02 | tsc --noEmit has no new errors | unit      | `npx tsc --noEmit 2>&1` (expect only pre-existing TS2307)                                                         | N/A                |

### Sampling Rate

- **Per task commit:** `npx tsc --noEmit` -- verify no new TypeScript errors introduced
- **Per wave merge:** Push to branch, verify all 5 CI workflows pass on GitHub Actions
- **Phase gate:** All CI workflows green + `tsc --noEmit` shows only pre-existing TS2307 error

### Wave 0 Gaps

None -- existing CI infrastructure covers all phase requirements. No new test files needed.

## Sources

### Primary (HIGH confidence)

- [Microsoft TypeScript Node Target Mapping](https://github.com/microsoft/TypeScript/wiki/Node-Target-Mapping) - Verified Node.js 24 maps to ES2024 target/lib
- [Node.js 24.14.0 LTS release](https://nodejs.org/en/blog/release/v24.14.0) - Latest LTS version confirmed as 24.14.0
- [npm registry @types/node](https://registry.npmjs.org/@types/node) - Latest 24.x version is 24.12.0
- [GitHub Actions metadata syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/metadata-syntax) - Confirms `using: 'node24'` is valid

### Secondary (MEDIUM confidence)

- [GitHub Changelog: Deprecation of Node 20](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/) - Timeline for Node 20 deprecation and Node 24 default
- [actions/checkout Node 24 PR](https://github.com/actions/checkout/pull/2226) - Runner v2.327.0 requirement for node24

### Tertiary (LOW confidence)

None -- all findings verified with primary sources.

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH - all versions verified against npm registry and official releases
- Architecture: HIGH - configuration changes are well-documented with exact line numbers in existing files
- Pitfalls: HIGH - pre-existing TS2307 error confirmed by running `tsc --noEmit` on current codebase; FORCE flag count confirmed by `git grep`

**Research date:** 2026-03-17
**Valid until:** 2026-04-17 (stable; Node.js LTS and TypeScript target mapping change infrequently)
