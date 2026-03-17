# Phase 3: Node.js 24 API Compatibility Audit

**Audited:** 2026-03-17
**Scope:** All Node.js built-in API usage in `nx-set-shas.ts` and `tools/pre-commit.ts`
**Node.js migration:** 20 -> 24

## Summary

Zero breaking changes found. All Node.js built-in APIs used in this codebase are stable across Node.js 20 through 24. No source code changes are required for Node.js 24 compatibility at the API level.

## Audit: nx-set-shas.ts

| API                    | File             | Line(s)                                                            | Node.js 24 Status                                                                             | Action Needed |
| ---------------------- | ---------------- | ------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- | ------------- |
| `spawnSync`            | `nx-set-shas.ts` | import: 4; usage: 40, 52, 63, 98, 103, 250                         | Stable, no breaking changes                                                                   | None          |
| `existsSync`           | `nx-set-shas.ts` | import: 5; usage: 30                                               | Stable; Node.js 24 adds runtime warning for invalid inputs, but codebase passes valid strings | None          |
| `process.chdir`        | `nx-set-shas.ts` | 31                                                                 | Stable, no breaking changes                                                                   | None          |
| `process.stdout.write` | `nx-set-shas.ts` | 33-34, 87-89, 111-113, 115-117, 122-124, 131-132, 143-149, 155-157 | Stable, no breaking changes                                                                   | None          |
| `process.env`          | `nx-set-shas.ts` | 12, 185                                                            | Stable, no breaking changes                                                                   | None          |

## Audit: tools/pre-commit.ts

| API            | File                  | Line(s)                          | Node.js 24 Status           | Action Needed |
| -------------- | --------------------- | -------------------------------- | --------------------------- | ------------- |
| `execSync`     | `tools/pre-commit.ts` | import: 1; usage: 13, 18, 26, 34 | Stable, no breaking changes | None          |
| `process.exit` | `tools/pre-commit.ts` | 7, 41                            | Stable, no breaking changes | None          |
| `process.env`  | `tools/pre-commit.ts` | 15, 21                           | Stable, no breaking changes | None          |

## Findings

### Pre-commit hook build command discrepancy

The pre-commit hook in `tools/pre-commit.ts` (line 13) runs `npm run build`, while the project's `package.json` build script uses `bun build`. Since `npm run build` delegates to the package.json script which invokes `bun build`, this works in practice but is an indirect invocation. The canonical command is `bun run build`.

**Status:** Noted as finding. Not fixed -- out of scope for this migration.

## Out-of-Scope Items

### 1. `/dev/null` git argument (line 105)

`nx-set-shas.ts` line 105 passes `/dev/null` as a literal argument to `git hash-object -t tree /dev/null`. This is a git CLI argument, not a Node.js API. On Windows runners without `/dev/null`, git may produce different behavior. This is a pre-existing cross-platform concern unrelated to the Node.js 20-to-24 migration.

### 2. `stripNewLineEndings()` latent bug (SRCF-01)

The `stripNewLineEndings()` function at line 280-282 uses `string.replace('\n', '')` which only replaces the first occurrence, not all newlines. This is a pre-existing logic issue, not a Node.js 24 regression.

### 3. stdio normalization (SRCF-02)

The `spawnSync` call at line 250-252 uses `stdio: ['pipe', 'pipe', null]` which sends stderr to the parent process. Other `spawnSync` calls use the default stdio. This inconsistency is pre-existing and unrelated to Node.js 24.

---

**Conclusion:** This codebase is fully compatible with Node.js 24 at the built-in API level. All APIs used (`spawnSync`, `execSync`, `existsSync`, `process.chdir`, `process.stdout.write`, `process.env`, `process.exit`) have no breaking changes between Node.js 20 and 24.
