# Research Summary: Node.js 24 Migration

**Project:** nrwl/nx-set-shas GitHub Action
**Domain:** GitHub Action runtime migration (node20 -> node24)
**Researched:** 2026-03-17
**Confidence:** HIGH

## Executive Summary

The nx-set-shas action needs to migrate its declared runtime from `node20` to `node24` in `action.yml` before GitHub removes Node.js 20 from runners in fall 2026 (with a default switch on June 2, 2026). The good news: this codebase uses an unusually narrow Node.js API surface — `child_process.spawnSync`, `fs.existsSync`, `process.env`, and `process.stdout.write` — none of which have breaking changes between Node.js 20 and 24. The migration is primarily a configuration change plus a handful of small code quality fixes that the upgrade exposes.

The recommended approach is minimal and surgical: change `action.yml` to `node24`, update `@types/node` to 24.x, bump the Volta pin and `engines` field, fix two known source code issues (`catch (e)` and `stripNewLineEndings`), rebuild `dist/`, and release as v5.0.0. The `@actions/core` and `@actions/github` dependencies should be kept at their current versions — they are pure CJS JavaScript with no native addons and will run on Node.js 24 without changes. Upgrading them introduces unnecessary risk (major version jumps to ESM-only packages) with no benefit.

The key risk is the transitive dependency `undici@5.29.0` bundled inside `@actions/github`. Node.js 24 ships with `undici@7` built-in, and the bundled v5 could exhibit subtle HTTP behavior differences. This risk is managed by validating with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` in CI before releasing. The overall migration complexity is LOW — this should be a single focused PR.

## Key Findings

### Recommended Stack

The codebase already uses the right tools. Bun bundles TypeScript into a single `dist/nx-set-shas.js` file targeting Node.js, so the module format of dependencies (CJS vs ESM) is irrelevant at runtime — everything gets bundled. This means the ESM-only `@actions/core@3.0.0` and `@actions/github@9.0.0` are not needed and would only introduce complexity.

**Core technologies and their migration posture:**

- `action.yml` `using: 'node20'` -> `'node24'`: The single required runtime change — one line edit
- `@types/node@^24.0.0`: Must update to get correct compile-time type checking for Node.js 24 APIs
- `volta.node` -> `24.12.0` (latest Node.js 24 LTS): Aligns developer environment with CI runtime
- `@actions/core@1.11.1`: Keep as-is — pure CJS JS, works on Node.js 24 without changes
- `@actions/github@6.0.1`: Keep as-is — deep import `@actions/github/lib/utils` would break on v9
- TypeScript 5.8.3: Keep as-is — fully supports the required target config
- Bun 1.2.19: Keep as-is — build tool, not runtime; output is Node-compatible regardless

### Expected Features

This is a migration, not a feature build. "Features" are the changes required for correct operation on the new runtime.

**Must do (migration blockers):**

- Change `action.yml` `using` from `node20` to `node24` — the core deliverable
- Update `@types/node` from 20.x to 24.x — enables compile-time verification of API compatibility
- Update Volta pin and `engines.node` to Node.js 24 — aligns dev environment with runtime
- Fix `catch (e)` at line 78 to guard `.message` access — exposed by stricter TypeScript types in `@types/node@24`
- Fix `stripNewLineEndings` to use `.trim()` instead of single-replace — pre-existing bug, risk increases with Node.js 24 stream behavior changes
- Rebuild `dist/nx-set-shas.js` — build artifact must be regenerated after any changes
- Bump version to 5.0.0 — runtime change is a breaking change for consumers (v4 -> v5)

**Should do (quality improvements enabled by migration):**

- Add `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true` to test workflows — validates compatibility before June 2026 deadline
- Normalize `stdio: ['pipe', 'pipe', null]` to `stdio: ['pipe', 'pipe', 'pipe']` at line 251 — defensive fix for stricter Node.js 24 validation
- Consider updating `tsconfig.json` target/lib from `ES2023` to `ES2024` — Node.js 24 (V8 13.6) supports ES2024 natively

**Defer (out of scope for this migration):**

- Upgrade `@actions/github` to 9.0.0 — three Octokit major versions, high break risk, no compatibility necessity
- Upgrade `@actions/core` to 3.0.0 — two major versions, ESM-only in latest, unnecessary risk
- Refactor `nx-set-shas.ts` architecture — out of scope per PROJECT.md
- Replace `process.stdout.write` with `core.info`/`core.warning` — enhancement, not migration work
- Fix `/dev/null` cross-platform issue at line 103 — pre-existing issue, hardcoded fallback hash masks it

### Architecture Approach

The migration follows a tiered sequence that matches the dependency graph: runtime declaration first, then type system, then source audit guided by compiler errors, then build and test, then version bump and release. This order ensures each tier reveals problems for the next tier before committing to later changes.

**Migration tiers in order:**

1. **Runtime declaration** (`action.yml`) — the primary deliverable; must be first
2. **Type system** (`@types/node`, `bun install`, `tsc --noEmit`) — surfaces any API incompatibilities at compile time before touching source
3. **Source code fixes** (`nx-set-shas.ts`) — guided by type errors from tier 2; fixes `catch (e)` and `stripNewLineEndings`
4. **Configuration updates** (`package.json` version, Volta, engines, optional `tsconfig.json`) — metadata that tracks the migration
5. **Build and test** (`bun run build`, CI with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`) — validation gate
6. **Release** (v5.0.0 tag, release notes) — consumer communication

### Critical Pitfalls

1. **Upgrading dependencies unnecessarily** — `@actions/core` and `@actions/github` work on Node.js 24 as-is. Only update if `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` testing reveals a concrete incompatibility. The research shows zero Node.js API breaking changes in this codebase.

2. **Skipping pre-migration baseline test** — Run the test workflow with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` BEFORE making any changes. If it passes, the migration is mostly config changes. If it fails, you know exactly what to fix. This is the most valuable single step.

3. **Stale `dist/` in release** — The action runs from `dist/nx-set-shas.js`, not the TypeScript source. Changes to source are invisible until `bun run build` regenerates the artifact. The pre-commit hook should catch this, but verify it is functional.

4. **Untyped `catch (e)` at line 78** — Accessing `e.message` on an `unknown`-typed catch variable is a TypeScript compile error under strict mode. This will surface when `@types/node` is updated. Fix with `e instanceof Error ? e.message : String(e)`.

5. **Self-hosted runner consumers need runner agent v2.327.1+** — Release notes must document this. Users on older self-hosted runner agents will see the action fail after the v5 upgrade, not because of the code, but because their runner agent does not support node24.

## Implications for Roadmap

Based on research, the migration maps cleanly to a 4-phase roadmap:

### Phase 1: Pre-Migration Baseline Validation

**Rationale:** Establish current compatibility before touching anything. If the action already works on Node.js 24 (likely), the migration is purely configuration. If not, the failure pinpoints exactly what needs fixing. This is the "measure twice" step.
**Delivers:** Confidence level for the migration; test workflow updated with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true`
**Addresses:** Pitfall — "Assuming current code is incompatible before testing"
**Avoids:** Wasted dependency upgrades triggered by phantom incompatibilities

### Phase 2: Configuration and Type System Changes

**Rationale:** Config changes carry zero risk and unlock the type-checking validation mechanism. `@types/node` must be updated before the source audit to let the compiler surface any API issues. These changes are logically atomic — they either all apply or none should.
**Delivers:** Updated `action.yml` (node24), updated `@types/node@^24.0.0`, updated Volta pin to Node.js 24 LTS, updated `engines.node` to `>=24`, clean `tsc --noEmit` pass
**Uses:** Official Node.js 24.12.0 LTS release
**Implements:** Tiers 1-2 from architecture migration sequence

### Phase 3: Source Code Fixes and Build

**Rationale:** Source fixes are guided by the type errors surfaced in Phase 2. The number of fixes is known and small (2-3 changes). Build regeneration must happen last in this phase so `dist/` reflects all source changes.
**Delivers:** Fixed `catch (e)` guard at line 78, fixed `stripNewLineEndings` using `.trim()`, normalized `stdio` at line 251, rebuilt `dist/nx-set-shas.js`
**Implements:** Tiers 3-5 from architecture migration sequence
**Avoids:** Shipping stale `dist/` artifact

### Phase 4: Version Bump and Release

**Rationale:** Version bump is last because it is the consumer-facing signal that changes are complete. Release notes must document self-hosted runner requirements and OpenSSL 3.5 implications for GHES users. The v4 tag must be preserved pointing to the last node20-compatible commit.
**Delivers:** package.json version 5.0.0, updated CHANGELOG/release notes, v5.0.0 tag, preserved v4 tag
**Addresses:** Consumer migration path (v4 -> v5), self-hosted runner documentation, GHES OpenSSL notes

### Phase Ordering Rationale

- **Baseline first** because it takes 10 minutes and may eliminate most of the work (the action might already pass on node24).
- **Config before code** because `@types/node@24` acts as a lint pass that identifies source issues; fixing config first means source audit is compiler-guided, not manual.
- **Build at end of Phase 3** (not Phase 2) because the build artifact must capture all source fixes atomically.
- **Version bump last** because bumping to 5.0.0 before tests pass would be premature. Keep the bump as a release gate.

### Research Flags

Phases with well-documented patterns (no additional research needed):

- **Phase 1:** `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` mechanism is officially documented; standard CI config change
- **Phase 2:** Config changes are mechanical; version numbers confirmed from npm registry and nodejs.org
- **Phase 3:** All source fixes are identified and confirmed in source code; no unknown surface area
- **Phase 4:** Standard GitHub Actions semver tagging practice; well-documented

No phases require additional research. The migration is fully characterized. The one area to watch during execution (not requiring upfront research) is whether `undici@5.29.0` behaves correctly on Node.js 24 — this is validated by Phase 1 testing, not by research.

## Confidence Assessment

| Area         | Confidence | Notes                                                                                                               |
| ------------ | ---------- | ------------------------------------------------------------------------------------------------------------------- |
| Stack        | HIGH       | Official Node.js migration guides verified; npm registry data for all deps; Bun bundler behavior confirmed          |
| Features     | HIGH       | Source code analyzed directly; all API call sites identified; no guesswork                                          |
| Architecture | HIGH       | Migration sequence based on actual dependency graph in the codebase; no speculative components                      |
| Pitfalls     | HIGH       | Most pitfalls confirmed from official GitHub/Node.js sources; source code issues confirmed at specific line numbers |

**Overall confidence:** HIGH

### Gaps to Address

- **`undici@5` on Node.js 24 HTTP behavior:** No official compatibility statement found. MEDIUM risk. Handled by Phase 1 baseline testing — if tests pass with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`, this gap is closed without code changes.
- **`@actions/core` 1.11.1 official Node.js 24 support statement:** No official compat matrix exists. MEDIUM confidence the 1.x line works (pure JS, no native deps). Closed by Phase 1 testing.
- **Node.js 24 LTS patch version:** Research identified 24.12.0 as latest LTS (December 2025). Confirm actual latest LTS at time of execution — a newer patch release may be available.

## Sources

### Primary (HIGH confidence)

- [GitHub Changelog: Deprecation of Node 20 on GitHub Actions runners](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/) — timeline, node22 skip, FORCE env var
- [Node.js Official: v20 to v22 Migration Guide](https://nodejs.org/en/blog/migrations/v20-to-v22) — breaking changes
- [Node.js Official: v22 to v24 Migration Guide](https://nodejs.org/en/blog/migrations/v22-to-v24) — breaking changes, OpenSSL 3.5
- [Node.js Official: v24.0.0 Release](https://nodejs.org/en/blog/release/v24.0.0) — V8 version, feature set
- [actions/toolkit RELEASES.md](https://github.com/actions/toolkit/blob/main/packages/core/RELEASES.md) — @actions/core version history
- [GitHub Runner Issue #4295](https://github.com/actions/runner/issues/4295) — FORCE flag deprecation warning bug
- [GitHub Runner Issue #4064](https://github.com/actions/runner/issues/4064) — self-hosted runner update issue
- Source code analysis of `nx-set-shas.ts` and `dist/nx-set-shas.js` — direct line-level verification

### Secondary (MEDIUM confidence)

- [NodeSource: Node.js 24 Becomes LTS](https://nodesource.com/blog/nodejs-24-becomes-lts) — LTS date and version
- [GitHub Community Discussion #160454](https://github.com/orgs/community/discussions/160454) — node22 skip confirmation
- [actions/setup-node Node 24 PR #1325](https://github.com/actions/setup-node/pull/1325) — migration pattern reference
- [FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 runner issue](https://github.com/actions/runner/issues/4295) — testing mechanism

### Tertiary (LOW confidence)

- tsconfig.json ES2024/ES2025 target update — optional improvement, no hard requirement identified

---

_Research completed: 2026-03-17_
_Ready for roadmap: yes_
