# Phase 4: Version Bump and Release PR - Context

**Gathered:** 2026-03-17
**Status:** Ready for planning

<domain>
## Phase Boundary

Package the Node.js 24 migration as v5.0.0 and submit a clean PR to the upstream `nrwl/nx-set-shas` repository. The current working branch has 36 commits including GSD planning artifacts that must not appear in the upstream PR. This phase extracts the 4 migration commits onto a clean branch, adds a version bump, verifies CI, and creates the PR.

</domain>

<decisions>
## Implementation Decisions

### Clean branch strategy

- Create branch `feat/node24-runtime` from `upstream/main` (fetch upstream fresh first)
- Cherry-pick exactly these 4 migration commits from the working branch:
  1. `feat(02-01): update runtime and toolchain to Node.js 24 / ES2024`
  2. `chore(02-01): remove FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 from CI workflows`
  3. `feat(02-02): bump @types/node from ^20.19.9 to ^24.12.0`
  4. `feat(03): bump @actions/core to 3.x and @actions/github to 9.x`
- Add a 5th commit: `chore: bump version to 5.0.0`
- Push to `origin/feat/node24-runtime`, open PR against `nrwl/nx-set-shas`
- Keep the current working branch (`LayZeeDK/feat/migrate-to-node24-runtime`) as a permanent dev record on the fork — do not delete it

### Version bump

- Bump `package.json` version from `4.4.0` to `5.0.0`
- Commit message: `chore: bump version to 5.0.0` (matches upstream convention)
- Let the pre-commit hook run — it will rebuild `dist/` and auto-stage it
- No manual rebuild needed; the hook produces an identical artifact (bundle does not embed version)

### dist/ artifact

- dist/ is already current from Phase 3 (pre-commit hook rebuilt it during the `@actions/*` upgrade commit)
- No rebuild required before the version bump commit
- The version bump commit's pre-commit hook will rebuild + stage dist/ automatically as part of normal commit flow

### PR title

- `feat!: update action runtime to node24`
- The `!` suffix signals breaking change per conventional commits

### PR description structure

Structured with these sections:

1. **Why** — Brief motivation: GitHub is deprecating node20 Actions on June 2, 2026; all consumers of `@v4` currently see deprecation warnings
2. **Breaking Changes** — Two explicit callouts: runtime change and self-hosted runner requirement
3. **Migrating from v4** — Concise guide: update tag + runner note
4. **Self-Hosted Runners** — Detailed callout with GitHub Actions Runner version, GHES note, ARM32 note
5. **Testing** — Links to CI runs for all four workflows (test, test-integration, format, publish)
6. **Closes** — `Closes #208`

### PR description — Breaking changes

Call out only two breaking changes (the others are internal implementation details for consumers using the action via `uses:`):

- **node20 → node24 runtime**: `action.yml` now declares `node24`
- **Self-hosted runner requirement**: GitHub Actions Runner v2.327.1+ required

### PR description — Migration guide

Concise format:

```yaml
# Before
uses: nrwl/nx-set-shas@v4

# After
uses: nrwl/nx-set-shas@v5
```

Self-hosted runner note inline.

### PR description — Self-hosted runner section

Key facts to state accurately (researched from actions/runner#3940):

- Requires **GitHub Actions Runner v2.327.1 or later** (released July 25, 2025)
- Node.js 24 is **bundled inside the GitHub Actions Runner** — no separate Node.js installation needed on the host machine
- Runners with auto-update enabled receive this automatically
- **GHES**: Older GHES instances may cap the runner version below v2.327.1 — verify GHES version supports GitHub Actions Runner >= v2.327.1
- **Linux ARM32 runners**: Not supported — Node.js 24 dropped 32-bit ARM support
- Do NOT say "Node.js 24 must be installed on the runner machine" — this is incorrect

### PR description — Testing section

Include links to CI runs for all four fork workflows passing:

- `test.yml` (cross-platform matrix: ubuntu, macOS, Windows)
- `test-integration.yml`
- `format.yml`
- `publish.yml`

### Claude's Discretion

- Exact wording and prose within each PR description section
- Whether to combine the version bump commit with dist/ rebuild into one commit if the pre-commit hook produces no diff
- How to verify CI on the clean branch before opening the PR
- Exact cherry-pick command flags (e.g., `-x` to reference original commits)

</decisions>

<specifics>
## Specific Ideas

- The upstream PR closes issue #208 on `nrwl/nx-set-shas`
- The runner self-hosted section must NOT say "install Node.js 24" — the correct requirement is GitHub Actions Runner v2.327.1+ (Node.js is bundled in the runner, not a host dependency)
- PR title uses `feat!` (breaking change marker) not `feat` — the runtime change is a hard breaking change for consumers
- CONTRIBUTING.md says: "simply update the version in package.json and merge into main" — the PR itself is the release mechanism; tags are applied automatically by the publish workflow after merge
- CONTRIBUTING.md also says to update dist/ if source files changed — the pre-commit hook handles this automatically
- The fork's working branch (`LayZeeDK/feat/migrate-to-node24-runtime`) documents the full migration process including planning artifacts — preserve as a permanent reference

</specifics>

<code_context>

## Existing Code Insights

### Reusable Assets

- `dist/nx-set-shas.js`: Already rebuilt from Phase 3 (`feat(03)` commit) — current and committed, no action needed before version bump
- Pre-commit hook (`tools/pre-commit.ts`): Runs `bun run build` and auto-stages `dist/` on every commit — handles the version bump commit automatically
- `publish.yml`: The upstream publish workflow applies `v5`, `v5.0`, and `v5.0.0` tags after merge — no manual tagging needed

### Established Patterns

- Upstream version bump convention: `chore: Bump version from X.X.X to Y.Y.Y (#PR)` — adapt to `chore: bump version to 5.0.0` without PR number (that goes in the merge commit)
- dist/ is always committed (checked-in convention, confirmed in CONTRIBUTING.md and Phase 3)
- PR to upstream is the release mechanism — no separate tagging or npm publish step

### Integration Points

- `upstream` remote: `https://github.com/nrwl/nx-set-shas.git` — already configured
- `origin` remote: `https://github.com/LayZeeDK/nrwl-nx-set-shas.git` — push clean branch here, open PR against upstream
- 4 commits to cherry-pick: identified by message prefix `feat(02-01)`, `chore(02-01)`, `feat(02-02)`, `feat(03)` from `main..HEAD` log

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

_Phase: 04-version-bump-and-release-pr_
_Context gathered: 2026-03-17_
