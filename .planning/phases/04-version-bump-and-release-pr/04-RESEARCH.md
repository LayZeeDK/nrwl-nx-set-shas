# Phase 4: Version Bump and Release PR - Research

**Researched:** 2026-03-17
**Domain:** Git branch management, GitHub Actions PR workflow, conventional commits
**Confidence:** HIGH

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions

**Clean branch strategy:**

- Create branch `feat/node24-runtime` from `upstream/main` (fetch upstream fresh first)
- Cherry-pick exactly these 4 migration commits from the working branch:
  1. `feat(02-01): update runtime and toolchain to Node.js 24 / ES2024`
  2. `chore(02-01): remove FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 from CI workflows`
  3. `feat(02-02): bump @types/node from ^20.19.9 to ^24.12.0`
  4. `feat(03): bump @actions/core to 3.x and @actions/github to 9.x`
- Add a 5th commit: `chore: bump version to 5.0.0`
- Push to `origin/feat/node24-runtime`, open PR against `nrwl/nx-set-shas`
- Keep the current working branch (`LayZeeDK/feat/migrate-to-node24-runtime`) as a permanent dev record on the fork — do not delete it

**Version bump:**

- Bump `package.json` version from `4.4.0` to `5.0.0`
- Commit message: `chore: bump version to 5.0.0` (matches upstream convention)
- Let the pre-commit hook run — it will rebuild `dist/` and auto-stage it
- No manual rebuild needed; the hook produces an identical artifact (bundle does not embed version)

**dist/ artifact:**

- dist/ is already current from Phase 3 (pre-commit hook rebuilt it during the `@actions/*` upgrade commit)
- No rebuild required before the version bump commit
- The version bump commit's pre-commit hook will rebuild + stage dist/ automatically as part of normal commit flow

**PR title:** `feat!: update action runtime to node24`

**PR description structure** (5 sections):

1. **Why** — GitHub is deprecating node20 Actions on June 2, 2026; all consumers of `@v4` currently see deprecation warnings
2. **Breaking Changes** — Two explicit callouts: runtime change and self-hosted runner requirement
3. **Migrating from v4** — Concise guide: update tag + runner note
4. **Self-Hosted Runners** — Detailed callout with GitHub Actions Runner version, GHES note, ARM32 note
5. **Testing** — Links to CI runs for all four workflows (test, test-integration, format, publish)
6. **Closes** — `Closes #208`

**PR breaking changes (two only):**

- node20 → node24 runtime: `action.yml` now declares `node24`
- Self-hosted runner requirement: GitHub Actions Runner v2.327.1+ required

**PR migration guide format:**

```yaml
# Before
uses: nrwl/nx-set-shas@v4

# After
uses: nrwl/nx-set-shas@v5
```

**Self-hosted runner section — key facts:**

- Requires GitHub Actions Runner v2.327.1 or later (released July 25, 2025)
- Node.js 24 is bundled inside the GitHub Actions Runner — no separate Node.js installation needed on the host machine
- Runners with auto-update enabled receive this automatically
- GHES: Older GHES instances may cap the runner version below v2.327.1 — verify GHES version supports GitHub Actions Runner >= v2.327.1
- Linux ARM32 runners: Not supported — Node.js 24 dropped 32-bit ARM support
- Do NOT say "Node.js 24 must be installed on the runner machine" — this is incorrect

**Testing section:** Links to CI runs for all four fork workflows passing (test.yml, test-integration.yml, format.yml, publish.yml)

### Claude's Discretion

- Exact wording and prose within each PR description section
- Whether to combine the version bump commit with dist/ rebuild into one commit if the pre-commit hook produces no diff
- How to verify CI on the clean branch before opening the PR
- Exact cherry-pick command flags (e.g., `-x` to reference original commits)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>

## Phase Requirements

| ID      | Description                                                                                       | Research Support                                                                                                                                                                         |
| ------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RLSE-01 | Bump package version from 4.4.0 to 5.0.0                                                          | Direct edit to `package.json` `"version"` field; pre-commit hook handles dist/ rebuild automatically                                                                                     |
| RLSE-02 | Commit rebuilt `dist/` (repo convention: built output is checked in)                              | Pre-commit hook in `tools/pre-commit.ts` runs `npm run build`, diffs dist/, and auto-stages changed files; no manual step needed                                                         |
| RLSE-03 | Verify fork CI passes (test, test-integration, format workflows use `./` and `GITHUB_TOKEN` only) | CI workflows confirmed: `test.yml` (ubuntu/macOS/Windows matrix), `test-integration.yml`, `format.yml` — all use `./` action reference and `${{ github.token }}` only; no secrets needed |
| RLSE-04 | Create PR against upstream `nrwl/nx-set-shas` with breaking changes documented in PR description  | Clean branch `feat/node24-runtime` from `upstream/main` + cherry-pick 4 commits + version bump; `gh pr create` targeting `nrwl/nx-set-shas:main`                                         |

</phase_requirements>

## Summary

Phase 4 is a packaging and delivery phase — all implementation work is already complete. The goal is to extract the four migration commits from the working branch (which contains 36+ commits including all GSD planning artifacts) onto a clean branch that is fit for upstream consumption, add a version bump commit, push to the fork, and open a PR against `nrwl/nx-set-shas`.

The critical constraint is branch cleanliness. The upstream PR must contain exactly 5 commits: 4 migration commits cherry-picked from the working branch and 1 version bump commit. No planning docs, no temp test commits, no GSD artifacts. The working branch `LayZeeDK/feat/migrate-to-node24-runtime` is preserved as a permanent dev record.

The pre-commit hook (`tools/pre-commit.ts`) is a key automation asset: it runs `npm run build` (which calls Bun), checks for dist/ changes, and auto-stages them. The version bump commit triggers this automatically, so no manual dist/ rebuild is needed before committing.

**Primary recommendation:** Create `feat/node24-runtime` from `upstream/main`, cherry-pick 4 specific commits by their exact SHAs, bump version, push, and open PR with `gh pr create`.

## Standard Stack

### Core Tools for This Phase

| Tool   | Version               | Purpose                            | Why Used                                                            |
| ------ | --------------------- | ---------------------------------- | ------------------------------------------------------------------- |
| git    | system                | Cherry-pick, branch management     | SCM; cherry-pick isolates exactly the commits needed                |
| gh CLI | system                | PR creation against upstream fork  | Native GitHub PR creation with body HEREDOC support                 |
| bun    | 1.2.19 (package.json) | Build dist/ during pre-commit hook | Project's build toolchain; called by pre-commit via `npm run build` |

### Commit SHAs to Cherry-Pick

Exact SHAs identified from `git log --format="%H %s" main..HEAD`:

| Commit SHA (short) | Full SHA                                   | Message                                                                     |
| ------------------ | ------------------------------------------ | --------------------------------------------------------------------------- |
| `f7ea1ff`          | `f7ea1ff1fd7c72e997473bc1d77e975a94df0174` | `feat(02-01): update runtime and toolchain to Node.js 24 / ES2024`          |
| `c3f3ceb`          | `c3f3ceb5bdc8242aea3c43af41761e94c456588d` | `chore(02-01): remove FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 from CI workflows` |
| `4ff1652`          | `4ff1652b60da795bbef765386a8b7eb055d85c1b` | `feat(02-02): bump @types/node from ^20.19.9 to ^24.12.0`                   |
| `54af1c8`          | `54af1c8e110b8b89bcd15081554953f0da5495d8` | `feat(03): bump @actions/core to 3.x and @actions/github to 9.x`            |

**Order matters:** Cherry-pick in the order shown above (chronological — earliest first).

### Git Remotes (Verified)

| Remote     | URL                                                | Role                              |
| ---------- | -------------------------------------------------- | --------------------------------- |
| `origin`   | `https://github.com/LayZeeDK/nrwl-nx-set-shas.git` | Fork — push clean branch here     |
| `upstream` | `https://github.com/nrwl/nx-set-shas.git`          | Source of truth — PR targets this |

**upstream/main verified fresh:** `3e9ad73 chore: Bump version from 4.3.3 to 4.4.0 (#204)` — same tip as fork `main`, no divergence.

## Architecture Patterns

### Clean Branch Workflow

```
upstream/main  ──────────────────────────────────────────── main
                    \
feat/node24-runtime  ──[cp1]──[cp2]──[cp3]──[cp4]──[v5]──> PR → nrwl/nx-set-shas

LayZeeDK/feat/migrate-to-node24-runtime  (preserved, 36+ commits, GSD artifacts)
```

Where:

- `[cp1..cp4]` = the 4 cherry-picked migration commits
- `[v5]` = `chore: bump version to 5.0.0` (triggers pre-commit hook → dist/ rebuild + format)

### Pattern 1: Branch Creation and Cherry-Pick

**What:** Create a clean branch from upstream/main, cherry-pick exactly 4 commits.

**Key decision for cherry-pick flags:** The `-x` flag appends `(cherry picked from commit <sha>)` to the commit message. This provides traceability back to the working branch. Whether to use it is Claude's discretion — the working branch is public on the fork, so `-x` is not strictly required but aids future forensics.

**Example flow:**

```bash
# Ensure upstream is fresh
git fetch upstream

# Create clean branch from upstream/main
git checkout -b feat/node24-runtime upstream/main

# Cherry-pick in chronological order
git cherry-pick f7ea1ff1fd7c72e997473bc1d77e975a94df0174
git cherry-pick c3f3ceb5bdc8242aea3c43af41761e94c456588d
git cherry-pick 4ff1652b60da795bbef765386a8b7eb055d85c1b
git cherry-pick 54af1c8e110b8b89bcd15081554953f0da5495d8
```

### Pattern 2: Version Bump with Pre-Commit Hook

**What:** Edit `package.json` version field, commit — pre-commit hook handles everything else.

**Pre-commit hook behavior (from `tools/pre-commit.ts`):**

1. Runs `npm run build` (= `bun build ./nx-set-shas.ts --outdir ./dist --target node`)
2. Checks `git diff --name-only` for files starting with `dist/`
3. Auto-stages any changed dist/ files via `git add <file>`
4. Runs `npm run format` (= `prettier --write .`)
5. Exits 0 on success — commit proceeds

**Important:** The pre-commit hook uses `npm run build`, not `bun run build`. On this machine, npm should delegate to bun (via `packageManager: "bun@1.2.19"`). Verify bun is available in PATH when the hook runs.

```bash
# Edit package.json version: "4.4.0" → "5.0.0"
# Then commit — hook fires automatically
git add package.json
git commit -m "chore: bump version to 5.0.0"
# Hook runs: builds dist/, stages changes, formats, completes
```

### Pattern 3: Push and Open PR

**What:** Push clean branch to origin, create PR targeting upstream.

```bash
git push -u origin feat/node24-runtime

gh pr create \
  --repo nrwl/nx-set-shas \
  --base main \
  --head LayZeeDK:feat/node24-runtime \
  --title "feat!: update action runtime to node24" \
  --body "$(cat <<'EOF'
[PR description here]
EOF
)"
```

**Note on `--repo` flag:** `gh pr create --repo <upstream>` targets the upstream repo. The `--head` must be qualified as `LayZeeDK:feat/node24-runtime` to reference the fork branch.

### Anti-Patterns to Avoid

- **Committing from the working branch directly:** The working branch has 36+ commits including docs/planning artifacts. Never use it as the PR branch.
- **Manual dist/ rebuild before version bump:** The pre-commit hook handles this. Running `bun run build` manually before committing the version bump is redundant and risks a double-build that may produce no diff, confusing the staging logic.
- **Using `git add .` or `git add -A`:** Forbidden by global CLAUDE.md. Always stage specific files.
- **Force-pushing to main or upstream:** The clean branch is `feat/node24-runtime` on origin — a new branch, so the initial push is always clean.
- **Skipping the pre-commit hook (`--no-verify`):** The hook is the mechanism that commits dist/. Skipping it means dist/ won't be updated, violating RLSE-02.

## Don't Hand-Roll

| Problem                          | Don't Build                                 | Use Instead                                                                                         | Why                                                              |
| -------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| PR creation                      | Custom API call via curl to GitHub REST API | `gh pr create`                                                                                      | gh CLI handles auth, fork detection, and cross-repo PRs natively |
| dist/ rebuild tracking           | Manual diff + stage loop                    | Pre-commit hook (`tools/pre-commit.ts`)                                                             | Already implemented; runs on every commit automatically          |
| Conventional commits conformance | Manual message formatting check             | PR title `feat!: update action runtime to node24` follows the pattern; upstream merge will apply it | Upstream commit history uses this format                         |

**Key insight:** This phase has no code to write. The "implementation" is git operations. Use the tools that already exist (git, gh CLI, the pre-commit hook) and don't introduce new scripting.

## Common Pitfalls

### Pitfall 1: Cherry-Pick Order

**What goes wrong:** Picking commits in reverse order causes compilation errors on intermediate commits (e.g., picking `feat(02-02): bump @types/node` before `feat(02-01): update runtime` creates a state where `@types/node` 24.x is referenced but `target`/`lib` in tsconfig still point to ES2023).

**Why it happens:** Each commit was built sequentially on top of the previous. Cherry-picking out of order creates a dependency mismatch.

**How to avoid:** Always cherry-pick in chronological order: 02-01 runtime → 02-01 CI cleanup → 02-02 types → 03 actions libs.

**Warning signs:** `tsc` errors during the pre-commit hook on a cherry-pick step.

### Pitfall 2: `upstream/main` Not Fresh

**What goes wrong:** If upstream has received a new commit since the last `git fetch upstream`, the clean branch diverges from the true upstream tip. The PR would show extraneous commits.

**Why it happens:** upstream/main may advance independently.

**How to avoid:** Always run `git fetch upstream` immediately before `git checkout -b feat/node24-runtime upstream/main`.

**Status:** Verified fresh as of 2026-03-17: tip is `3e9ad73 chore: Bump version from 4.3.3 to 4.4.0 (#204)`.

### Pitfall 3: Pre-Commit Hook Fails on Clean Branch

**What goes wrong:** The pre-commit hook requires `bun` and `npm` in PATH. On the clean branch, `node_modules/` is not installed yet. The hook calls `npm run build` which invokes a bun-based build.

**Why it happens:** `git checkout -b feat/node24-runtime upstream/main` gives a clean tree. Cherry-picked commits include a `package.json` with bun as packageManager, but `bun install` has not been run.

**How to avoid:** Run `bun install` on the clean branch before committing the version bump. The cherry-pick commits don't trigger the hook's build step until a commit is made — but the version bump commit does.

**Warning signs:** `Cannot find module` or `command not found: bun` errors from the hook.

### Pitfall 4: PR Description Misstates Runner Requirements

**What goes wrong:** Saying "install Node.js 24 on your self-hosted runner" — this is incorrect and causes user confusion.

**Why it happens:** Natural assumption that runtime = OS dependency.

**How to avoid:** Node.js 24 is **bundled inside GitHub Actions Runner v2.327.1+**. The only requirement is runner version, not a host Node.js installation. Source: actions/runner#3940.

**Warning signs:** Any sentence containing "install Node.js 24 on your runner" in the PR description draft.

### Pitfall 5: format.yml Only Triggers on PRs to Main (Not on Push)

**What goes wrong:** After pushing `feat/node24-runtime` to origin, `format.yml` won't run on push — it only runs on `pull_request` targeting `main`. The PR to upstream is the only way to see format CI pass.

**Why it happens:** `format.yml` has `on: pull_request: branches: [main]` — it does not have a `push` trigger.

**How to avoid:** `test.yml` and `test-integration.yml` run on push (they have both `push` and `pull_request` triggers, with `paths-ignore: ["**.md"]`). Format is only visible after the PR is open. Verify formatting locally with `bun run format:check` before pushing.

### Pitfall 6: `gh pr create` Fork Branch Reference

**What goes wrong:** Running `gh pr create --repo nrwl/nx-set-shas` from the fork may default `--head` to the local branch name without the fork owner qualifier, resulting in "branch not found" error on the upstream.

**Why it happens:** GitHub's PR API requires `owner:branch` when the head is on a fork different from the base repo.

**How to avoid:** Always explicitly pass `--head LayZeeDK:feat/node24-runtime` when targeting the upstream.

## Code Examples

### Version Field Edit (package.json)

```json
{
  "version": "5.0.0"
}
```

Only this one field changes. No other package.json fields need updating.

### Full Cherry-Pick Sequence

```bash
git fetch upstream
git checkout -b feat/node24-runtime upstream/main
bun install
git cherry-pick f7ea1ff1fd7c72e997473bc1d77e975a94df0174
git cherry-pick c3f3ceb5bdc8242aea3c43af41761e94c456588d
git cherry-pick 4ff1652b60da795bbef765386a8b7eb055d85c1b
git cherry-pick 54af1c8e110b8b89bcd15081554953f0da5495d8
```

### Version Bump Commit

```bash
# Edit package.json: "version": "4.4.0" → "5.0.0"
git add package.json
git commit -m "chore: bump version to 5.0.0"
# Pre-commit hook fires: builds dist/, stages dist/ if changed, runs prettier
```

### PR Creation Command

```bash
git push -u origin feat/node24-runtime

gh pr create \
  --repo nrwl/nx-set-shas \
  --base main \
  --head LayZeeDK:feat/node24-runtime \
  --title "feat!: update action runtime to node24" \
  --body "$(cat <<'EOF'
## Why

GitHub is [deprecating node20 Actions on June 2, 2026](https://github.blog/changelog/2024-03-07-github-actions-all-actions-will-run-on-node20-instead-of-node16-by-default/). All consumers of `@v4` currently see deprecation warnings in their workflow runs.

## Breaking Changes

- **Runtime**: `action.yml` now declares `node24` (was `node20`)
- **Self-hosted runners**: Requires [GitHub Actions Runner v2.327.1 or later](https://github.com/actions/runner/releases/tag/v2.327.1)

## Migrating from v4

\`\`\`yaml
# Before
uses: nrwl/nx-set-shas@v4

# After
uses: nrwl/nx-set-shas@v5
\`\`\`

Self-hosted runner users: see section below.

## Self-Hosted Runners

Node.js 24 is **bundled inside the GitHub Actions Runner** — no separate Node.js installation is required on the host machine.

Requirements:

- **GitHub Actions Runner v2.327.1 or later** (released July 25, 2025)
- Runners with auto-update enabled receive this automatically
- **GHES**: Verify your GHES version supports GitHub Actions Runner >= v2.327.1
- **Linux ARM32**: Not supported — Node.js 24 dropped 32-bit ARM support

## Testing

All four fork CI workflows pass on this branch:

- `test.yml` — ubuntu-latest, macOS-latest, windows-latest
- `test-integration.yml` — ubuntu-latest, macOS-latest, windows-latest
- `format.yml`
- `publish.yml`

[CI run links to be added after push]

Closes #208
EOF
)"
```

### CI Verification Before Opening PR

```bash
# Format check locally (format.yml won't run on push)
bun run format:check

# Confirm test.yml and test-integration.yml were triggered (check origin fork Actions tab)
# URL: https://github.com/LayZeeDK/nrwl-nx-set-shas/actions
```

## State of the Art

| Old Approach                   | Current Approach       | Notes           |
| ------------------------------ | ---------------------- | --------------- |
| `node20` runtime in action.yml | `node24` runtime       | Done in Phase 2 |
| Version 4.4.0                  | Version 5.0.0          | This phase      |
| Working branch with docs       | Clean 5-commit branch  | This phase      |
| No upstream PR                 | PR to nrwl/nx-set-shas | This phase      |

**Upstream publish mechanism (from CONTRIBUTING.md and publish.yml):**

- No manual tagging needed — `publish.yml` fires on merge to `upstream/main`
- It calls `jameshenry/publish-shell-action@v1` which applies `v5`, `v5.0`, `v5.0.0` tags automatically
- The PR itself is the release mechanism

## Open Questions

1. **Will cherry-picked commits conflict with upstream/main?**
   - What we know: upstream/main tip (`3e9ad73`) was verified fresh. The 4 migration commits each touch different files from what upstream has changed recently.
   - What's unclear: Can't confirm zero conflicts until cherry-pick is attempted.
   - Recommendation: If conflicts arise, resolve manually keeping the migration intent. Most likely files: `action.yml`, `package.json`, `tsconfig.json`.

2. **Will the pre-commit hook run correctly on the clean branch during cherry-pick?**
   - What we know: Cherry-pick doesn't trigger the pre-commit hook. Only the version bump commit triggers it.
   - What's unclear: Whether `bun install` must be run before any cherry-pick or only before the version bump commit.
   - Recommendation: Run `bun install` after the last cherry-pick, before the version bump commit. This ensures node_modules is present for the hook.

3. **Will `format.yml` pass given the cherry-picked commits were formatted by prettier during their original commit?**
   - What we know: The pre-commit hook ran `npm run format` on each original commit. The prettier config is unchanged.
   - What's unclear: Whether the prettier version pinned in devDependencies produces identical output across machines/environments.
   - Recommendation: Run `bun run format:check` locally before pushing. If it fails, run `bun run format` and amend the version bump commit.

## Validation Architecture

### Test Framework

| Property           | Value                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------ |
| Framework          | GitHub Actions CI (no local unit test framework)                                                       |
| Config file        | `.github/workflows/test.yml`, `.github/workflows/test-integration.yml`, `.github/workflows/format.yml` |
| Quick run command  | `bun run format:check` (local)                                                                         |
| Full suite command | Push to `origin/feat/node24-runtime` and monitor GitHub Actions                                        |

### Phase Requirements to Test Map

| Req ID  | Behavior                                             | Test Type   | Automated Command                                                                                                   | File Exists?                                                          |
| ------- | ---------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| RLSE-01 | `package.json` version == `5.0.0`                    | smoke       | `node -e "const p=require('./package.json'); if(p.version!=='5.0.0') throw new Error('wrong version: '+p.version)"` | N/A (inline)                                                          |
| RLSE-02 | `dist/nx-set-shas.js` committed and current          | smoke       | `git status --short -- dist/` outputs nothing (clean)                                                               | N/A (git check)                                                       |
| RLSE-03 | Fork CI passes (test, test-integration, format)      | integration | Push to origin, observe GitHub Actions                                                                              | ✅ `.github/workflows/test.yml`, `test-integration.yml`, `format.yml` |
| RLSE-04 | PR exists against upstream with breaking change docs | manual      | `gh pr list --repo nrwl/nx-set-shas --head LayZeeDK:feat/node24-runtime`                                            | N/A (PR check)                                                        |

### Sampling Rate

- **Per task commit:** `bun run format:check` + `git status --short -- dist/`
- **Per wave merge:** N/A (single wave)
- **Phase gate:** All GitHub Actions green on `origin/feat/node24-runtime` before opening PR; PR opened and URL confirmed

### Wave 0 Gaps

None — existing CI infrastructure covers all phase requirements. No new test files needed.

## Sources

### Primary (HIGH confidence)

- Direct file inspection: `package.json` — confirmed current version `4.4.0`, bun build script, volta Node.js 24.14.0
- Direct file inspection: `action.yml` — confirmed `runs: using: 'node24'` (Phase 2 already applied)
- Direct file inspection: `tools/pre-commit.ts` — confirmed auto-rebuild and auto-stage behavior
- Direct file inspection: `.husky/pre-commit` — confirmed hook calls `bunx lint-staged` then `bun tools/pre-commit.ts`
- Direct file inspection: `.github/workflows/test.yml`, `test-integration.yml`, `format.yml`, `publish.yml`
- Direct inspection: `git log --format="%H %s" main..HEAD` — 4 migration commit SHAs confirmed
- Direct inspection: `git remote -v` — origin and upstream remotes confirmed
- Direct inspection: `git fetch upstream && git log upstream/main` — upstream/main fresh at `3e9ad73`
- CONTRIBUTING.md — confirmed "update version in package.json and merge into main" release process

### Secondary (MEDIUM confidence)

- CONTEXT.md (04-CONTEXT.md) — actions/runner#3940 referenced for self-hosted runner v2.327.1 requirement and bundled Node.js fact

### Tertiary (LOW confidence)

- None required — all critical claims verified from local file inspection

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH — all tooling verified from local files (git, gh, bun, pre-commit hook)
- Architecture: HIGH — commit SHAs confirmed via git log, remote URLs verified, branch flow modeled from CONTEXT.md decisions
- Pitfalls: HIGH (most) / MEDIUM (cherry-pick conflicts) — hook behavior confirmed from source; conflict risk is inherent to git operations

**Research date:** 2026-03-17
**Valid until:** 2026-04-17 (stable toolchain; upstream/main staleness is only risk — re-fetch before executing)
