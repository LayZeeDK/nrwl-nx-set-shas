# Roadmap: nx-set-shas Node.js 24 Migration

## Overview

Migrate the nrwl/nx-set-shas GitHub Action from the deprecated Node.js 20 runtime to Node.js 24, released as v5.0.0. The migration follows a measure-first approach: validate current compatibility on Node.js 24, update configuration and type system, fix any surfaced issues and rebuild, then bump version and submit PR to upstream.

## Phases

**Phase Numbering:**

- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Baseline Validation** - Verify current action behavior on Node.js 24 before making changes
- [ ] **Phase 2: Configuration and Type System** - Update runtime declaration, type definitions, and toolchain config for Node.js 24
- [ ] **Phase 3: Source Fixes and Build** - Fix compiler-surfaced issues and rebuild dist/ artifact
- [ ] **Phase 4: Version Bump and Release PR** - Bump to v5.0.0 and submit PR to upstream

## Phase Details

### Phase 1: Baseline Validation

**Goal**: Establish whether the action already works on Node.js 24 before touching any code
**Depends on**: Nothing (first phase)
**Requirements**: BVAL-02, BVAL-03
**Success Criteria** (what must be TRUE):

1. CI test workflow runs with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` and results are recorded
2. Baseline pass/fail status is documented so Phase 2-3 scope is informed by evidence, not assumptions
   **Plans**: 1 plan

Plans:

- [x] 01-01-PLAN.md -- Bump action deps to node24-native versions, add FORCE flag to all workflows, run CI, and record baseline results

### Phase 2: Configuration and Type System

**Goal**: All project configuration points to Node.js 24 and the TypeScript compiler surfaces any API incompatibilities
**Depends on**: Phase 1
**Requirements**: RUNT-01, RUNT-02, RUNT-03, TSCO-01, TSCO-02, TYPE-01, TYPE-02
**Success Criteria** (what must be TRUE):

1. `action.yml` declares `node24` as the runtime
2. `package.json` Volta pin and engines field require Node.js 24
3. `tsconfig.json` target and lib match the TypeScript Node Target Mapping for Node.js 24
4. `@types/node` is at 24.x and `tsc --noEmit` either passes clean or surfaces only known fixable errors
   **Plans**: 2 plans

Plans:

- [ ] 02-01-PLAN.md -- Update runtime declaration (action.yml node24), toolchain config (package.json, tsconfig.json), and remove FORCE flag from all 5 CI workflows
- [ ] 02-02-PLAN.md -- Bump @types/node to 24.x, update lockfile, and verify tsc --noEmit produces no new errors

### Phase 3: Source Fixes and Build

**Goal**: All source code compiles cleanly on Node.js 24 types and the built dist/ artifact reflects all changes
**Depends on**: Phase 2
**Requirements**: AUDT-01, AUDT-02, AUDT-03, BVAL-01
**Success Criteria** (what must be TRUE):

1. All Node.js API usage in `nx-set-shas.ts` and `tools/pre-commit.ts` is audited and documented
2. `tsc --noEmit` passes with zero errors on `@types/node` 24.x
3. `bun run build` produces a fresh `dist/nx-set-shas.js` that reflects all source changes
4. CI tests pass on Node.js 24 runtime
   **Plans**: 2 plans

Plans:

- [ ] 03-01-PLAN.md -- Audit Node.js 24 API compatibility and upgrade @actions/core to 3.x, @actions/github to 9.x
- [ ] 03-02-PLAN.md -- Rebuild dist/nx-set-shas.js and verify CI green on all platforms

### Phase 4: Version Bump and Release PR

**Goal**: The migration is packaged as v5.0.0 and submitted to upstream for merge
**Depends on**: Phase 3
**Requirements**: RLSE-01, RLSE-02, RLSE-03, RLSE-04
**Success Criteria** (what must be TRUE):

1. `package.json` version is 5.0.0
2. `dist/nx-set-shas.js` is committed and up to date with source
3. All fork CI workflows (test, test-integration, format) pass
4. PR exists against `nrwl/nx-set-shas` with breaking changes and self-hosted runner requirements documented in description
   **Plans**: TBD

Plans:

- [ ] 04-01: Bump version, verify CI, and create upstream PR

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4

| Phase                            | Plans Complete | Status      | Completed  |
| -------------------------------- | -------------- | ----------- | ---------- |
| 1. Baseline Validation           | 1/1            | Complete    | 2026-03-17 |
| 2. Configuration and Type System | 1/2            | In Progress |            |
| 3. Source Fixes and Build        | 1/2            | In Progress |            |
| 4. Version Bump and Release PR   | 0/1            | Not started | -          |
