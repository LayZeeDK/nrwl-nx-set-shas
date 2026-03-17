# Phase 2: Configuration and Type System - Context

**Gathered:** 2026-03-17
**Status:** Ready for planning

<domain>
## Phase Boundary

Update all project configuration to target Node.js 24: runtime declaration in action.yml, Volta pin and engines in package.json, TypeScript target/lib in tsconfig.json, and @types/node version. Fix any TypeScript compilation errors surfaced by the type definition upgrade. Remove the temporary FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 flag added in Phase 1 (now redundant with native node24 declaration).

</domain>

<decisions>
## Implementation Decisions

### TypeScript target mapping

- Set both `target` and `lib` to `ES2024`, per the official [Microsoft TypeScript Node Target Mapping](https://github.com/microsoft/TypeScript/wiki/Node-Target-Mapping) for Node.js 24
- Current values are `ES2023` (matching Node.js 20) -- this is a one-step bump
- `module` stays `nodenext` (unchanged)

### Node.js 24 version pin

- Volta pin: set to latest Node.js 24.x LTS version available at plan time (researcher resolves exact version)
- `engines.node`: bump from `>=20` to `>=24`
- This is a breaking change (contributors must have Node.js 24) -- acceptable since the entire migration is a major version bump

### TEMP flag cleanup

- Remove `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` from all three CI workflows (`test.yml`, `integration-test-workflow.yml`, `format.yml`) in this phase
- The flag was added in Phase 1 (commit `1cc440b`) as `[TEMP]` and is redundant once `action.yml` declares `node24`
- Remove the flag in the same commit or immediately after the `action.yml` runtime change

### Type error strategy

- Bump `@types/node` from `^20.x` to `^24.x`
- Run `tsc --noEmit` after the bump
- Fix any compilation errors inline in Phase 2 -- the build must be green after config changes
- Phase 3 then handles deeper source-level audit for runtime API deprecations that tsc cannot catch

### Claude's Discretion

- Exact Node.js 24.x version for Volta pin (resolve latest LTS at research time)
- Commit granularity within the phase (single commit vs separate config/types commits)
- Order of config file changes
- How to verify tsc passes after changes

</decisions>

<specifics>
## Specific Ideas

- The official TypeScript Node Target Mapping wiki was explicitly chosen over community forks -- use [microsoft/TypeScript/wiki/Node-Target-Mapping](https://github.com/microsoft/TypeScript/wiki/Node-Target-Mapping) as the authoritative reference
- Phase 1 confirmed zero source changes needed on Node.js 24, so type errors from @types/node 24.x are expected to be minimal or nonexistent

</specifics>

<code_context>

## Existing Code Insights

### Reusable Assets

- `action.yml` line 45: `using: 'node20'` -- change to `'node24'`
- `package.json` lines 16-22: `engines`, `volta`, `packageManager` -- update node version fields
- `tsconfig.json`: 7-line file with `target`, `lib`, `module` -- straightforward edit
- `package.json` line 29: `@types/node` dependency -- bump version range

### Established Patterns

- Bun as build tool (`bun build ./nx-set-shas.ts --outdir ./dist --target node`)
- TypeScript type-checking via `tsc --noEmit` (not part of build, but validates types)
- Pre-commit hook in `tools/pre-commit.ts` runs build and format

### Integration Points

- `action.yml` runtime declaration is consumed by GitHub Actions runner
- `volta.node` pin is consumed by Volta-aware development environments
- `engines.node` is consumed by npm/bun for compatibility warnings
- `@types/node` version affects all TypeScript source files (`nx-set-shas.ts`, `tools/pre-commit.ts`)
- CI workflows (`test.yml`, `integration-test-workflow.yml`, `format.yml`) have the FORCE flag to remove

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

_Phase: 02-configuration-and-type-system_
_Context gathered: 2026-03-17_
