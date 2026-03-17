# Requirements: nx-set-shas Node.js 24 Migration

**Defined:** 2026-03-17
**Core Value:** The action runs without deprecation warnings on GitHub Actions Node.js 24 runtime and all existing functionality works correctly.

## v1 Requirements

Requirements for this migration. Each maps to roadmap phases.

### Runtime Configuration

- [ ] **RUNT-01**: Update `action.yml` runtime declaration from `node20` to `node24`
- [ ] **RUNT-02**: Update Volta Node.js pin in `package.json` to Node.js 24.x
- [ ] **RUNT-03**: Update `engines.node` in `package.json` to require Node.js >= 24

### TypeScript Configuration

- [ ] **TSCO-01**: Update `tsconfig.json` `target` per TypeScript Node Target Mapping for Node.js 24
- [ ] **TSCO-02**: Update `tsconfig.json` `lib` per TypeScript Node Target Mapping for Node.js 24

### Type System

- [ ] **TYPE-01**: Update `@types/node` from 20.x to 24.x
- [ ] **TYPE-02**: Fix all TypeScript compilation errors surfaced by `@types/node` 24.x (e.g., `catch (e)` clause typing)

### Source Code Audit

- [ ] **AUDT-01**: Audit all Node.js API usage in `nx-set-shas.ts` for Node.js 24 compatibility
- [ ] **AUDT-02**: Audit all Node.js API usage in `tools/pre-commit.ts` for Node.js 24 compatibility
- [ ] **AUDT-03**: Document audit findings (APIs checked, changes needed, no-change confirmations)

### Build & Validation

- [ ] **BVAL-01**: Rebuild `dist/nx-set-shas.js` with Bun targeting Node.js 24
- [ ] **BVAL-02**: All existing CI tests pass on Node.js 24 runtime
- [ ] **BVAL-03**: Validate action works end-to-end with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true`

### Version & Release

- [ ] **RLSE-01**: Bump package version from 4.4.0 to 5.0.0
- [ ] **RLSE-02**: Commit rebuilt `dist/` (repo convention: built output is checked in)
- [ ] **RLSE-03**: Verify fork CI passes (test, test-integration, format workflows use `./` and `GITHUB_TOKEN` only)
- [ ] **RLSE-04**: Create PR against upstream `nrwl/nx-set-shas` with breaking changes documented in PR description

## v2 Requirements

Deferred to future work. Not in current migration scope.

### Source Fixes

- **SRCF-01**: Fix `stripNewLineEndings()` latent bug (single-string `replace` instead of `.trim()`)
- **SRCF-02**: Normalize stdio options in spawnSync calls

### Dependency Updates

- **DEPD-01**: Update `@actions/core` to latest major version
- **DEPD-02**: Update `@actions/github` to latest major version

### TypeScript Configuration

- **TSCO-03**: Evaluate updating `tsconfig.json` `module` setting per Node Target Mapping

## Out of Scope

| Feature                                | Reason                                                  |
| -------------------------------------- | ------------------------------------------------------- |
| Build toolchain modernization          | Keep Bun as-is; not related to Node.js 24 migration     |
| Dependency updates unrelated to node24 | Minimal scope; only update if compatibility requires it |
| Backwards compatibility with node20    | Clean v5 break, no dual support                         |
| New features or refactoring            | Migration only                                          |
| Tagging/publishing v5 release          | This is a fork; release means PR to upstream            |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase   | Status  |
| ----------- | ------- | ------- |
| RUNT-01     | Phase 2 | Pending |
| RUNT-02     | Phase 2 | Pending |
| RUNT-03     | Phase 2 | Pending |
| TSCO-01     | Phase 2 | Pending |
| TSCO-02     | Phase 2 | Pending |
| TYPE-01     | Phase 2 | Pending |
| TYPE-02     | Phase 2 | Pending |
| AUDT-01     | Phase 3 | Pending |
| AUDT-02     | Phase 3 | Pending |
| AUDT-03     | Phase 3 | Pending |
| BVAL-01     | Phase 3 | Pending |
| BVAL-02     | Phase 1 | Done    |
| BVAL-03     | Phase 1 | Done    |
| RLSE-01     | Phase 4 | Pending |
| RLSE-02     | Phase 4 | Pending |
| RLSE-03     | Phase 4 | Pending |
| RLSE-04     | Phase 4 | Pending |

**Coverage:**

- v1 requirements: 17 total
- Mapped to phases: 17
- Unmapped: 0

---

_Requirements defined: 2026-03-17_
_Last updated: 2026-03-17 after roadmap creation_
