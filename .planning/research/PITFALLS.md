# Pitfalls Research: Node.js 24 Migration

**Domain:** GitHub Action runtime migration (node20 -> node24)
**Researched:** 2026-03-17
**Overall confidence:** HIGH (official Node.js migration guide + GitHub changelog + codebase analysis)

## Critical Pitfalls

### Pitfall 1: `@actions/core` major version jump may be unnecessary and risky

**What goes wrong:** Multiple migration guides recommend upgrading `@actions/core` from 1.x to 2.x or 3.x as part of the node24 migration. However, the current `@actions/core` 1.11.1 is pure JavaScript with no native addons and no Node.js-version-gated APIs. Upgrading it introduces new transitive dependencies (`@actions/exec`, `@actions/http-client` v3/v4) that change behavior unrelated to the node24 migration.

**Why it happens:** Community migration templates copy-paste dependency bumps from other actions (e.g., `actions/checkout`, `actions/setup-node`) that had other reasons to upgrade. The `@actions/core` 2.x/3.x releases added features (OIDC, summary API, exec integration) but the breaking changes are in transitive deps, not the core API surface this action uses.

**Warning signs:** Build succeeds but runtime behavior changes in `getInput`, `setOutput`, `setFailed`, or `exportVariable` due to internal refactoring. Octokit instantiation via `@actions/github` changes subtly.

**Prevention:**

1. Test `@actions/core` 1.11.1 on Node.js 24 first -- it will almost certainly work without changes
2. Only upgrade if a concrete incompatibility is found
3. If upgrading, pin to exact version and test every API call used: `getInput`, `getBooleanInput`, `setOutput`, `exportVariable`, `setFailed`

**Detection:** Run the existing test workflow with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` using the current dependency versions before changing anything.

**Phase:** Pre-migration validation. Test current code on node24 runtime BEFORE changing dependencies.

**Confidence:** HIGH -- based on npm registry analysis of `@actions/core` 1.11.1 (pure JS, no native deps) and PROJECT.md constraint "only update if compatibility requires it."

---

### Pitfall 2: Untyped catch clause (`catch (e)`) breaks under strict TypeScript with `@types/node` 24.x

**What goes wrong:** Line 78 of `nx-set-shas.ts` has `catch (e)` followed by `e.message`. With `@types/node@24.x` and TypeScript 5.8+ strict mode, the caught error is typed as `unknown` by default (TypeScript's `useUnknownInCatchVariables` or strict mode). Accessing `.message` on `unknown` is a compile error.

**Why it happens:** `@types/node@20.x` -> `@types/node@24.x` does not directly cause this, but TypeScript strict mode (which may be enabled or become enabled) treats `catch` variables as `unknown`. The real trigger is that the current code has an implicit `any` catch parameter that works with lax settings but fails under stricter compilation.

**Warning signs:** TypeScript compilation fails with `Object is of type 'unknown'` at the `catch (e)` block.

**Prevention:**

```typescript
// Change from:
catch (e) {
  core.setFailed(e.message);
}
// To:
catch (e) {
  core.setFailed(e instanceof Error ? e.message : String(e));
}
```

**Phase:** Code audit and fix phase. Address during the TypeScript compilation pass.

**Confidence:** HIGH -- visible in source code at line 78, confirmed TypeScript behavior.

---

### Pitfall 3: `stripNewLineEndings()` only strips first newline -- exposed by Node.js 24 behavioral changes

**What goes wrong:** The `stripNewLineEndings` function on line 281 uses `string.replace('\n', '')` which only removes the FIRST newline character. If `spawnSync` output contains trailing `\r\n` (Windows) or multiple newlines, SHAs will contain leftover whitespace. While this bug exists today, Node.js 24's stricter stream/pipe behavior and potential differences in `spawnSync` stdout buffering could change the exact output format.

**Why it happens:** `String.replace()` with a string argument (not regex) only replaces the first match. This is a JavaScript language behavior, not a Node.js version issue, but the consequences become worse if output format changes.

**Warning signs:** `git` commands fail with "bad revision" errors because SHAs contain hidden whitespace characters.

**Prevention:** Replace with `.trim()`:

```typescript
function stripNewLineEndings(string: string): string {
  return string.trim();
}
```

**Phase:** Code audit phase. Fix as part of the compatibility audit.

**Confidence:** HIGH -- confirmed in source code analysis and CONCERNS.md.

---

### Pitfall 4: `/dev/null` in git command argument on Windows runners

**What goes wrong:** Line 103-106 passes `/dev/null` as an argument to `git hash-object -t tree /dev/null`. On GitHub-hosted Ubuntu runners this works fine. On Windows runners, `/dev/null` is translated by Git Bash but the behavior depends on how `spawnSync` invokes git. If the runner's git does not translate `/dev/null`, the command fails and the fallback hash computation breaks silently (the code falls back to the hardcoded hash).

**Why it happens:** The hardcoded `/dev/null` path is POSIX-specific. GitHub Actions runners are predominantly Linux, so this has worked. But Windows runner usage is increasing, and the empty tree hash computation silently fails.

**Warning signs:** On Windows runners, `git hash-object -t tree /dev/null` returns a non-zero exit code or unexpected output. The hardcoded fallback hash masks the failure.

**Prevention:** Use a cross-platform approach:

```typescript
// Platform-agnostic empty tree hash
const emptyTreeRes = spawnSync(
  'git',
  ['hash-object', '-t', 'tree', '--stdin'],
  {
    encoding: 'utf-8',
    input: '',
  },
);
```

Or simply use the well-known empty tree hash directly since it is a git constant.

**Phase:** Code audit phase. Low priority since the hardcoded fallback is correct, but should be cleaned up.

**Confidence:** MEDIUM -- the hardcoded fallback masks this issue, and most GA runners are Linux. But it is a latent bug.

---

### Pitfall 5: OpenSSL 3.5 security level change breaks HTTPS connections with older TLS configurations

**What goes wrong:** Node.js 24 ships OpenSSL 3.5 with security level 2 as default. This prohibits RSA/DSA/DH keys shorter than 2048 bits and ECC keys shorter than 224 bits. If any GitHub Enterprise Server instance or proxy in the runner's network path uses weak certificates, Octokit HTTPS calls will fail with cryptic TLS errors.

**Why it happens:** OpenSSL 3.5 raises the minimum key strength. This is invisible in development but manifests in production when runners connect through corporate proxies or to GitHub Enterprise with older certificates.

**Warning signs:** `UNABLE_TO_VERIFY_LEAF_SIGNATURE`, `ERR_TLS_CERT_ALTNAME_INVALID`, or `DEPTH_ZERO_SELF_SIGNED_CERT` errors during API calls that previously worked on node20.

**Prevention:**

1. This action only connects to api.github.com (public) which uses strong certs -- no issue for standard usage
2. Document in release notes that GHES users with custom certificates may be affected
3. Do NOT work around by lowering OpenSSL security level (`--openssl-legacy-provider` or `NODE_OPTIONS=--openssl-legacy-provider`)

**Phase:** Documentation/release notes phase. Not a code change -- a consumer-facing warning.

**Confidence:** HIGH -- official Node.js 24 migration guide confirms this change. LOW risk for this specific action (public GitHub API only).

## Common Mistakes

### Mistake 1: Updating all dependencies "while we're at it"

**What goes wrong:** Maintainers use the node24 migration as an excuse to bump every dependency, introducing unrelated breaking changes. When something breaks, it is impossible to tell if the issue is node24 or a dependency update.

**Why it happens:** It feels efficient to modernize everything at once. Package managers show outdated packages and it feels wrong to leave them.

**Prevention:** The PROJECT.md already constrains this correctly: "Update deps only when they block Node.js 24 compatibility." Enforce this by:

1. First: change ONLY `action.yml` from `node20` to `node24` and run tests
2. Second: update `@types/node` to 24.x and fix compilation errors
3. Third: update other deps ONLY if step 1 or 2 revealed incompatibilities

**Detection:** If the diff touches dependency versions that are not `@types/node`, question whether each change is actually required for node24.

**Phase:** Every phase. This is a process discipline, not a one-time fix.

**Confidence:** HIGH -- universally observed pattern in open-source migrations.

---

### Mistake 2: Forgetting to rebuild `dist/` after code changes

**What goes wrong:** The `action.yml` points to `dist/nx-set-shas.js`, not the TypeScript source. If source changes are made but `bun build` is not run, the published action runs stale code. The node24 runtime change in `action.yml` takes effect, but code fixes do not.

**Why it happens:** The pre-commit hook (`tools/pre-commit.ts`) should catch this, but if commits are made with `--no-verify` or the hook fails silently, stale dist files ship.

**Warning signs:** Tests pass locally (running source) but the action fails in CI (running dist).

**Prevention:**

1. Always run `npm run build` (which runs `bun build`) before testing
2. Verify the pre-commit hook is functional after any toolchain changes
3. Compare `dist/nx-set-shas.js` file size and modification time before and after changes

**Phase:** Build verification phase. Should be the final step before any release.

**Confidence:** HIGH -- the codebase already has a pre-commit hook for this, confirming it is a known risk.

---

### Mistake 3: Not testing with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` before release

**What goes wrong:** Maintainers change `action.yml` to `node24`, push to a release branch, and discover issues only when consumers report failures. There is no pre-release validation.

**Why it happens:** The node24 runtime is not yet the default on runners, so simply running CI does not exercise the node24 runtime unless explicitly forced.

**Warning signs:** CI passes on the migration PR but the action breaks for consumers after release.

**Prevention:**

1. Add `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true` to the test workflow environment BEFORE changing `action.yml`
2. This forces the runner to use node24 even while `action.yml` still says `node20`
3. If tests pass, the runtime is compatible; then change `action.yml`

**Phase:** Pre-migration validation. This should be the very first step.

**Confidence:** HIGH -- documented by GitHub as the official testing mechanism.

---

### Mistake 4: Version bump without updating Volta/engine configuration

**What goes wrong:** `package.json` has `"volta": { "node": "20.19.4" }` and `"engines": { "node": ">=20" }`. After migration, contributors still develop on Node.js 20 locally, potentially missing node24-specific issues. The mismatch between development runtime and CI runtime causes "works on my machine" bugs.

**Why it happens:** Volta and engine configs are easy to overlook since they do not affect the GitHub Actions runtime directly.

**Warning signs:** Local development uses node20 (per Volta), CI uses node24. Issues only appear in CI.

**Prevention:**

1. Update `volta.node` to a Node.js 24.x version
2. Update `engines.node` to `">=24"`
3. Ensure all contributors have Node.js 24 available locally

**Phase:** Configuration update phase. Should be done alongside the `action.yml` change.

**Confidence:** HIGH -- visible in package.json, confirmed in PROJECT.md requirements.

## GitHub Actions-Specific Gotchas

### Gotcha 1: No `node22` runtime option exists -- you must jump directly to `node24`

**What goes wrong:** Maintainers attempt a conservative migration to node22 first, discover that `runs.using: 'node22'` is not a valid option, and waste time. GitHub Actions explicitly skipped node22 support.

**Why it happens:** Node.js 22 is the current LTS and seems like a logical stepping stone. But GitHub decided to skip it entirely.

**Prevention:** Go directly from `node20` to `node24` in `action.yml`. There is no intermediate step.

**Phase:** Planning. This should be understood before starting.

**Confidence:** HIGH -- confirmed in GitHub changelog: "GitHub Actions is not planning to support a node22 option."

Sources:

- [GitHub Changelog: Deprecation of Node 20](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/)

---

### Gotcha 2: Deprecation warnings persist even after migration when using `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`

**What goes wrong:** When testing with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` while `action.yml` still says `node20`, the runner forces node24 but STILL emits the deprecation warning. This is confusing and makes it hard to verify the migration is complete.

**Why it happens:** Known bug in the GitHub Actions runner (issue #4295). The warning is based on the `action.yml` declaration, not the actual runtime used.

**Warning signs:** CI logs show "Node.js 20 actions are deprecated" even when you know the action is running on node24.

**Prevention:** Ignore the warning during the testing phase. The warning will disappear once `action.yml` is changed to `node24`. Verify actual runtime with `process.version` logging if needed.

**Phase:** Testing phase. Informational only.

**Confidence:** HIGH -- confirmed in [GitHub runner issue #4295](https://github.com/actions/runner/issues/4295).

---

### Gotcha 3: Self-hosted runners require manual update to support node24

**What goes wrong:** Organizations using self-hosted runners find that the action fails after migration because their runner agent is too old to support node24. The runner auto-update mechanism has bugs with the node20->node24 transition.

**Why it happens:** Runner agent version 2.327.1+ is required for node24 support. Older agents (e.g., 2.321.0) do not properly self-update to handle node24, per [runner issue #4064](https://github.com/actions/runner/issues/4064).

**Warning signs:** Action fails on self-hosted runners but works on GitHub-hosted runners.

**Prevention:**

1. Document in release notes that self-hosted runners need agent v2.327.1+
2. This action's consumers are primarily on GitHub-hosted runners, but the release notes should mention this

**Phase:** Documentation/release notes phase.

**Confidence:** HIGH -- confirmed in GitHub runner issues.

---

### Gotcha 4: Major version bump (v4 -> v5) requires consumer action

**What goes wrong:** Maintainers publish the node24 migration as a patch or minor release. Consumers on `@v4` get the node24 version automatically. If there are any behavioral differences, consumers break without opting in.

**Why it happens:** It feels like "just a runtime change" so a major bump seems excessive. But the runtime change IS a breaking change -- consumers who pin to specific Node.js behavior expectations will be affected.

**Prevention:**

1. Release as v5.0.0 (already planned per PROJECT.md)
2. Update the `v5` major version tag
3. Keep `v4` pointing to the last node20-compatible release
4. Consumers migrate by changing `nrwl/nx-set-shas@v4` to `nrwl/nx-set-shas@v5`

**Phase:** Release phase.

**Confidence:** HIGH -- standard GitHub Actions versioning practice.

---

### Gotcha 5: `spawnSync` stdio configuration with `null` may behave differently

**What goes wrong:** Line 251 of `nx-set-shas.ts` uses `stdio: ['pipe', 'pipe', null]` in the `commitExists` function's `spawnSync` call. The `null` for stderr means "inherit from parent process." Node.js 24's stricter validation in child_process could potentially treat `null` differently than node20 did.

**Why it happens:** The `null` value for stdio channels is semi-documented. Node.js 24 has been tightening validation across APIs.

**Warning signs:** `spawnSync` throws or warns about invalid stdio configuration.

**Prevention:**

```typescript
// More explicit and safe:
spawnSync('git', ['cat-file', '-e', commitSha], {
  stdio: ['pipe', 'pipe', 'pipe'],
});
```

**Phase:** Code audit phase.

**Confidence:** LOW -- no confirmed breaking change here, but the `null` stdio value is unusual and worth normalizing defensively.

## Prevention Checklist

### Before Starting Migration

- [ ] Run existing test workflow with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true` env variable to baseline current compatibility
- [ ] Read the official [Node.js v22 to v24 migration guide](https://nodejs.org/en/blog/migrations/v22-to-v24)
- [ ] Confirm no `node22` runtime exists -- migration must go directly to `node24`
- [ ] Review current `@actions/core` 1.11.1 and `@actions/github` 6.0.1 for node24 compatibility (likely compatible as-is)

### During Migration

- [ ] Change `action.yml` from `node20` to `node24` as the FIRST code change
- [ ] Update `@types/node` from 20.x to 24.x and fix ALL compilation errors
- [ ] Fix untyped `catch (e)` clause at line 78 (type as `unknown`, guard `.message` access)
- [ ] Fix `stripNewLineEndings` to use `.trim()` instead of single-replace
- [ ] Normalize `stdio: ['pipe', 'pipe', null]` to `stdio: ['pipe', 'pipe', 'pipe']`
- [ ] Do NOT update `@actions/core` or `@actions/github` unless a concrete incompatibility is found
- [ ] Update `volta.node` to Node.js 24.x version
- [ ] Update `engines.node` to `">=24"`
- [ ] Bump `version` from 4.4.0 to 5.0.0

### Before Release

- [ ] Run `npm run build` and verify `dist/nx-set-shas.js` is updated
- [ ] Verify pre-commit hook catches stale dist files
- [ ] Test on GitHub-hosted runners with actual workflow events (push, pull_request, merge_group)
- [ ] Document breaking changes for consumers in release notes
- [ ] Note self-hosted runner minimum version requirement (v2.327.1+)
- [ ] Note potential OpenSSL 3.5 impact for GHES users with custom certificates

### After Release

- [ ] Tag as v5.0.0 and update v5 major version tag
- [ ] Keep v4 tag pointing to last node20-compatible commit
- [ ] Monitor issues for unexpected runtime differences

## Phase-Specific Warnings

| Phase Topic            | Likely Pitfall                         | Mitigation                                                |
| ---------------------- | -------------------------------------- | --------------------------------------------------------- |
| Pre-validation         | Assuming current code is incompatible  | Test with `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` first |
| Dependency updates     | Updating `@actions/core` unnecessarily | Only update if concrete incompatibility found             |
| TypeScript compilation | `catch (e)` typed as `unknown`         | Guard with `instanceof Error` check                       |
| Code audit             | `stripNewLineEndings` bug exposed      | Replace with `.trim()`                                    |
| Code audit             | `stdio: null` in spawnSync             | Normalize to `'pipe'`                                     |
| Build                  | Stale `dist/` shipped                  | Verify pre-commit hook, run build explicitly              |
| Testing                | False deprecation warnings             | Known runner bug, ignore during testing                   |
| Release                | Patch/minor release instead of major   | Must be v5.0.0 per semver                                 |
| Consumer docs          | Self-hosted runner incompatibility     | Document runner v2.327.1+ requirement                     |
| Consumer docs          | OpenSSL 3.5 TLS failures on GHES       | Document in release notes                                 |

## Sources

- [Node.js v22 to v24 Migration Guide](https://nodejs.org/en/blog/migrations/v22-to-v24) -- HIGH confidence
- [GitHub Changelog: Deprecation of Node 20 on GitHub Actions runners](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/) -- HIGH confidence
- [GitHub Community Discussion on Node 20 Deprecation](https://github.com/orgs/community/discussions/189324) -- MEDIUM confidence
- [GitHub Runner Issue #4295: Deprecation warning with FORCE flag](https://github.com/actions/runner/issues/4295) -- HIGH confidence
- [GitHub Runner Issue #4064: Self-hosted runner update failures](https://github.com/actions/runner/issues/4064) -- HIGH confidence
- [Node.js userland-migrations issue #239](https://github.com/nodejs/userland-migrations/issues/239) -- HIGH confidence
- [NodeSource: Node.js 24 Becomes LTS](https://nodesource.com/blog/nodejs-24-becomes-lts) -- MEDIUM confidence

---

_Pitfalls research: 2026-03-17_
