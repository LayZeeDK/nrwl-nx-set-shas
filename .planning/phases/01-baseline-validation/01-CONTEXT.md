# Phase 1: Baseline Validation - Context

**Gathered:** 2026-03-17
**Status:** Ready for planning

<domain>
## Phase Boundary

Verify current action behavior on Node.js 24 before making any source code changes. Bump workflow action dependencies to their latest node24-native major versions, add the `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` env var to CI workflows, run the full test matrix, and record pass/fail results. This evidence scopes how much fixing Phases 2-3 actually need.

</domain>

<decisions>
## Implementation Decisions

### Workflow scope

- Add `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` to ALL three CI workflows: `test.yml`, `integration-test-workflow.yml`, and `format.yml`
- Full repo migration to Node.js 24 (not just the action runtime), so all workflows are in scope
- Run on all three platforms: ubuntu, macOS, Windows (existing matrix unchanged)
- Set the flag at workflow-level `env:` block (top-level, applies to all jobs/steps)

### Action dependency bumps

- Bump ALL `uses:` action dependencies to their latest major version with native `node24` declared in their `action.yml`
- Always take the latest major version, even if it has breaking changes requiring workflow adaptations
- Audit all `uses:` references across all workflow files to build the complete dependency list (known: `actions/checkout`, `oven-sh/setup-bun`, `jameshenry/publish-shell-action`)
- Research versions via GitHub Releases, CHANGELOG.md files, `v<N>` fluid major version tags, and/or commit history
- For `publish.yml`: include if `jameshenry/publish-shell-action` has a node24-compatible version; otherwise note it for the upstream PR description and leave it to the Nx team (James Henry is on the Nx team)
- Investigate and adapt to any breaking changes in the new major versions

### Commit strategy

- Separate commits: dep bumps first, then FORCE flag addition
- The FORCE flag commit must be easily droppable/revertable before the upstream PR
- FORCE flag commit message format: `test: [TEMP] add FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 for baseline validation`

### Flag permanence

- The FORCE flag is temporary -- only needed while our `action.yml` still declares `node20`
- Remove the flag in Phase 2, once `action.yml` is updated to declare `node24`
- If any dependency lacks a node24-native version, keep the FORCE flag per-workflow for those specific workflows until resolved

### Verification

- Verify that the FORCE flag is actually working (action ran under Node.js 24, not 20)
- Verification method: Claude's discretion during planning (note: `node --version` in `run:` steps shows PATH node, not the version used by `uses:` action steps)

### Results recording

- Create a dedicated markdown file: `.planning/phases/01-baseline-validation/01-BASELINE-RESULTS.md`
- Include a workflow x platform matrix table showing pass/fail per combination
- Include CI run link(s)
- Include failure details (error messages, relevant log excerpts) if anything fails
- Record which versions of action dependencies were used (exact versions, for reproducibility)
- Include a summary conclusion informing Phase 2-3 scope

### Claude's Discretion

- Exact verification method for confirming Node.js 24 was used by action steps
- Workflow-level `env:` placement details
- How to handle any edge cases in dependency version research

</decisions>

<code_context>

## Existing Code Insights

### Reusable Assets

- `action.yml`: Currently declares `node20` runtime (line 45) -- this is what the FORCE flag overrides
- `.github/workflows/test.yml`: Primary test workflow with PR and push event paths, cross-platform matrix
- `.github/workflows/integration-test-workflow.yml`: Reusable workflow called by `test-integration.yml`, tests action with `working-directory` input
- `.github/workflows/format.yml`: Prettier check workflow -- no action execution but validates toolchain on Node.js 24

### Established Patterns

- All test workflows use `uses: ./` to test the action against itself
- Cross-platform matrix: `[ubuntu-latest, macos-latest, windows-latest]` with `fail-fast: false`
- Bun as package manager and build tool (`bun install`, `bun run build`)
- Verification via inline Node.js scripts checking `NX_BASE` and `NX_HEAD` env vars

### Integration Points

- `test.yml` and `integration-test-workflow.yml` both compile and execute the action
- `test-integration.yml` calls `integration-test-workflow.yml` as a reusable workflow
- `publish.yml` uses `jameshenry/publish-shell-action@v1` (may need node24 update)

</code_context>

<specifics>
## Specific Ideas

- Before the final upstream PR, CI must pass both WITH and WITHOUT the FORCE flag -- the flag is only a Phase 1 validation tool
- Action dependency versions should be the latest major, not minimum node24-compatible -- deliver a modern, up-to-date codebase to upstream
- James Henry (publish-shell-action author) is on the Nx team, so if that action lacks node24 support, note it in the upstream PR for them to handle

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

_Phase: 01-baseline-validation_
_Context gathered: 2026-03-17_
