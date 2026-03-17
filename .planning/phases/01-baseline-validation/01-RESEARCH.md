# Phase 1: Baseline Validation - Research

**Researched:** 2026-03-17
**Domain:** GitHub Actions CI workflow configuration, Node.js 24 runtime forcing, action dependency management
**Confidence:** HIGH

## Summary

Phase 1 requires adding `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` to CI workflows and bumping action dependencies to their latest node24-native major versions. Research confirms the FORCE flag is a well-documented GitHub Actions runner feature that overrides `using: node20` declarations at runtime, forcing actions to execute under Node.js 24 instead. The flag is set as a workflow-level `env:` and applies to all jobs and steps within that workflow.

The critical finding is that **workflow-level `env:` variables do NOT propagate to reusable workflows called via `workflow_call`**. Since `test-integration.yml` calls `integration-test-workflow.yml` as a reusable workflow, the FORCE flag must be set inside `integration-test-workflow.yml` itself, not just in the caller. This is the primary implementation pitfall.

Action dependency research reveals: `actions/checkout` has v6 (node24-native, released 2025-11-20), `oven-sh/setup-bun` v2 already uses node24 natively on main, and `jameshenry/publish-shell-action` v1 still uses `node12` with no newer version available.

**Primary recommendation:** Set `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` at the top-level `env:` block in all four workflow files (`test.yml`, `format.yml`, `test-integration.yml`, AND `integration-test-workflow.yml`), bump `actions/checkout` to `v6` and keep `oven-sh/setup-bun` at `v2`, leave `publish-shell-action` at `v1` with a note for the upstream PR.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions

- Add `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` to ALL three CI workflows: `test.yml`, `integration-test-workflow.yml`, and `format.yml`
- Full repo migration to Node.js 24 (not just the action runtime), so all workflows are in scope
- Run on all three platforms: ubuntu, macOS, Windows (existing matrix unchanged)
- Set the flag at workflow-level `env:` block (top-level, applies to all jobs/steps)
- Bump ALL `uses:` action dependencies to their latest major version with native `node24` declared in their `action.yml`
- Always take the latest major version, even if it has breaking changes requiring workflow adaptations
- Audit all `uses:` references across all workflow files to build the complete dependency list
- For `publish.yml`: include if `jameshenry/publish-shell-action` has a node24-compatible version; otherwise note it for the upstream PR description
- Separate commits: dep bumps first, then FORCE flag addition
- The FORCE flag commit must be easily droppable/revertable before the upstream PR
- FORCE flag commit message format: `test: [TEMP] add FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 for baseline validation`
- The FORCE flag is temporary -- only needed while `action.yml` still declares `node20`
- Remove the flag in Phase 2, once `action.yml` is updated to declare `node24`
- Verify that the FORCE flag is actually working (action ran under Node.js 24, not 20)
- Create a dedicated markdown file: `.planning/phases/01-baseline-validation/01-BASELINE-RESULTS.md`
- Include a workflow x platform matrix table showing pass/fail per combination
- Include CI run link(s), failure details, dependency versions used, and summary conclusion

### Claude's Discretion

- Exact verification method for confirming Node.js 24 was used by action steps
- Workflow-level `env:` placement details
- How to handle any edge cases in dependency version research

### Deferred Ideas (OUT OF SCOPE)

None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID      | Description                                                                   | Research Support                                                                                                                                                                 |
| ------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BVAL-02 | All existing CI tests pass on Node.js 24 runtime                              | FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true env var forces node24 runtime for all action steps; must be set in each workflow file individually (not inherited by reusable workflows) |
| BVAL-03 | Validate action works end-to-end with FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true | Same mechanism; verification via runner logs showing node24 binary path in verbose output or deprecation warning absence                                                         |

</phase_requirements>

## Standard Stack

### Core

This phase involves no library installations -- it is purely CI workflow configuration.

| Tool                                 | Version          | Purpose              | Why Standard                                                         |
| ------------------------------------ | ---------------- | -------------------- | -------------------------------------------------------------------- |
| `actions/checkout`                   | v6.0.2           | Git checkout in CI   | Latest major with native `node24` runtime; released 2025-11-20       |
| `oven-sh/setup-bun`                  | v2.2.0           | Install Bun in CI    | Already declares `node24` in action.yml on main branch               |
| `jameshenry/publish-shell-action`    | v1.0.0           | Publish tagging      | Only version; still uses `node12` -- no node24-native release exists |
| `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` | Runner v2.327.1+ | Force node24 runtime | Official GitHub mechanism for pre-migration testing                  |

### Action Dependency Audit (Complete)

All `uses:` references across all workflow files:

| Workflow                      | Action                                              | Current | Latest node24-native  | Bump?                            |
| ----------------------------- | --------------------------------------------------- | ------- | --------------------- | -------------------------------- |
| test.yml                      | `actions/checkout`                                  | v4      | v6.0.2 (node24)       | YES -- v4 to v6                  |
| test.yml                      | `oven-sh/setup-bun`                                 | v2      | v2.2.0 (node24)       | NO -- already v2, already node24 |
| test.yml                      | `./` (self)                                         | node20  | n/a (FORCE overrides) | No bump needed                   |
| integration-test-workflow.yml | `actions/checkout`                                  | v4      | v6.0.2 (node24)       | YES -- v4 to v6                  |
| integration-test-workflow.yml | `./` (self)                                         | node20  | n/a (FORCE overrides) | No bump needed                   |
| test-integration.yml          | `./.github/workflows/integration-test-workflow.yml` | n/a     | n/a (reusable)        | n/a                              |
| format.yml                    | `actions/checkout`                                  | v4      | v6.0.2 (node24)       | YES -- v4 to v6                  |
| format.yml                    | `oven-sh/setup-bun`                                 | v2      | v2.2.0 (node24)       | NO -- already v2, already node24 |
| publish.yml                   | `actions/checkout`                                  | v4      | v6.0.2 (node24)       | YES -- v4 to v6                  |
| publish.yml                   | `jameshenry/publish-shell-action`                   | v1      | v1.0.0 (node12!)      | NO -- no newer version exists    |

### Alternatives Considered

| Instead of                     | Could Use             | Tradeoff                                                                                                  |
| ------------------------------ | --------------------- | --------------------------------------------------------------------------------------------------------- |
| `actions/checkout@v6`          | `actions/checkout@v5` | v5 is also node24-native but v6 has improved credential security; user decision says "latest major" so v6 |
| Bumping `publish-shell-action` | Forking it            | Out of scope; James Henry is on Nx team, note in upstream PR                                              |

## Architecture Patterns

### FORCE Flag Placement

The `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` environment variable must be set at the **workflow-level** `env:` block (top of the YAML file, before `jobs:`):

```yaml
# Source: GitHub Changelog 2025-09-19
name: 'Test'

on:
  push:
    paths-ignore:
      - '**.md'

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

jobs:
  test:
    # ...
```

### Reusable Workflow Env Propagation (CRITICAL)

Workflow-level `env:` variables do **NOT** propagate to reusable workflows called via `workflow_call`. This means:

- Setting `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` in `test-integration.yml` does **NOT** pass it to `integration-test-workflow.yml`
- The flag must be set **inside** `integration-test-workflow.yml` itself
- This is a GitHub Actions platform limitation, not configurable

The user's CONTEXT.md lists three workflows: `test.yml`, `integration-test-workflow.yml`, and `format.yml`. The flag must also be in `test-integration.yml` (the caller) for any actions it uses directly, though currently `test-integration.yml` only calls the reusable workflow and has no direct action steps.

### Commit Strategy Pattern

```
Commit 1: chore(ci): bump action dependencies to node24-native versions
  - actions/checkout v4 -> v6 in all workflow files
  - Handle any v6 breaking changes

Commit 2: test: [TEMP] add FORCE_JAVASCRIPT_ACTIONS_TO_NODE24 for baseline validation
  - Add env block to test.yml, integration-test-workflow.yml, format.yml
  - This commit is designed to be dropped before the upstream PR
```

### Anti-Patterns to Avoid

- **Setting FORCE flag only in caller workflows:** Reusable workflows do NOT inherit env vars from callers. The flag must be in each workflow file that contains action steps.
- **Using `node --version` in `run:` steps to verify:** `run:` steps use the PATH node (installed by setup-bun or setup-node), not the runner's internal node used for action execution. These are completely different binaries.
- **Bumping publish-shell-action beyond v1:** There is no v2; the only release is v1.0.0 using node12. The FORCE flag will upgrade it to node24 at runtime.

## Don't Hand-Roll

| Problem                   | Don't Build                   | Use Instead                                               | Why                                                        |
| ------------------------- | ----------------------------- | --------------------------------------------------------- | ---------------------------------------------------------- |
| Forcing node24 runtime    | Custom node version switching | `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` env var         | Official GitHub runner mechanism, handles all action types |
| Verifying node24 was used | Complex log parsing scripts   | Check runner verbose logs or deprecation warning behavior | Runner logs show which node binary was invoked             |

## Common Pitfalls

### Pitfall 1: Reusable Workflow Env Isolation

**What goes wrong:** Setting FORCE flag in `test-integration.yml` and expecting it to apply inside `integration-test-workflow.yml`
**Why it happens:** GitHub Actions reusable workflows run in an isolated context; they do not inherit the caller's `env:` block
**How to avoid:** Set the FORCE flag directly in `integration-test-workflow.yml`'s top-level `env:` block
**Warning signs:** Action still runs under node20 in integration tests despite flag being "set"

### Pitfall 2: Misleading Deprecation Warnings

**What goes wrong:** CI shows "actions are running on Node.js 20" deprecation annotations even when FORCE flag is active
**Why it happens:** Known runner bug (actions/runner#4295) -- the deprecation check reads the action.yml `using:` field before the FORCE override takes effect
**How to avoid:** Ignore these annotations; they are cosmetic. Verify actual runtime through other means.
**Warning signs:** Seeing deprecation warnings and incorrectly concluding the flag is not working

### Pitfall 3: actions/checkout v6 Credential Changes

**What goes wrong:** Git authentication fails in certain edge cases after bumping to v6
**Why it happens:** v6 changed credential persistence from `.git/config` headers to `includeIf.gitdir` mechanism in a separate file under `$RUNNER_TEMP`
**How to avoid:** For standard GitHub-hosted runners, v6 should work transparently. Known issues only affect: non-GitHub runners (Forgejo/Gitea), Git worktrees (fixed in v6.0.1), and Docker container actions
**Warning signs:** `fatal: could not read Username` errors in git operations after checkout

### Pitfall 4: paths-ignore Blocks .md Workflow Changes

**What goes wrong:** Pushing workflow changes alongside only `.md` file changes causes `test.yml` and `test-integration.yml` to not trigger
**Why it happens:** These workflows have `paths-ignore: ["**.md"]` -- if the commit ONLY touches `.md` files, the workflows are skipped
**How to avoid:** The workflow YAML files themselves are not `.md` files, so changes to `.github/workflows/*.yml` will trigger the workflows. Only an issue if the commit exclusively modifies markdown files.
**Warning signs:** CI workflows not appearing on the PR

### Pitfall 5: publish-shell-action on node12

**What goes wrong:** `publish.yml` uses `jameshenry/publish-shell-action@v1` which declares `node12`
**Why it happens:** The action has never been updated beyond v1.0.0
**How to avoid:** The FORCE flag will override node12 to node24. This may or may not work correctly depending on the action's code. Since `publish.yml` only runs on push to main and this is a fork, it is low risk. Note the situation in the baseline results for the upstream PR.
**Warning signs:** Publish workflow failures after node24 forcing

## Code Examples

### Workflow-Level FORCE Flag

```yaml
# Apply to each workflow file individually
name: 'Test'

on:
  push:
    paths-ignore:
      - '**.md'
  pull_request:
    paths-ignore:
      - '**.md'

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

concurrency:
  group: ${{ github.workflow }}-${{ github.event.number || github.ref }}
  cancel-in-progress: true

jobs:
  # ... existing jobs unchanged
```

### actions/checkout v6 Usage (Drop-in Replacement for v4)

```yaml
# v4 -> v6 is transparent for standard usage
# No input changes required for this repo's usage patterns
- uses: actions/checkout@v6
  name: Checkout [Pull Request]
  if: ${{ github.event_name == 'pull_request' }}
  with:
    ref: ${{ github.event.pull_request.head.sha }}
    fetch-depth: 0
    filter: tree:0
```

The `with:` inputs used in this repo (`ref`, `fetch-depth`, `filter`) are all preserved across v4/v5/v6 with no changes.

### Verification Method (Claude's Discretion)

The recommended verification approach: check the GitHub Actions runner logs for the action execution lines. When FORCE is active, the runner logs show the node24 binary path being used. Additionally:

1. **Positive signal:** If the action runs successfully and produces correct NX_BASE/NX_HEAD values, it is working on node24 (since FORCE is set).
2. **Negative signal (known bug):** Deprecation annotations saying "running on Node.js 20" should be IGNORED -- they fire based on the declared version, not the actual runtime (actions/runner#4295).
3. **Explicit check (optional):** Enable `ACTIONS_RUNNER_DEBUG: true` for one run to get verbose runner logs showing which node binary was invoked for each action step.

## State of the Art

| Old Approach                    | Current Approach                          | When Changed                | Impact                                                             |
| ------------------------------- | ----------------------------------------- | --------------------------- | ------------------------------------------------------------------ |
| `actions/checkout@v4` (node20)  | `actions/checkout@v6` (node24)            | v5: Aug 2025, v6: Nov 2025  | v6 adds credential isolation; transparent upgrade for standard use |
| `oven-sh/setup-bun@v2` (node20) | `oven-sh/setup-bun@v2` (node24)           | Updated on main             | Same major version, runtime updated in-place                       |
| No forced runtime               | `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` | Runner v2.327.1 (Fall 2025) | Enables pre-migration testing                                      |
| node20 default                  | node24 default                            | June 2, 2026 (upcoming)     | FORCE flag becomes unnecessary after this date                     |

**Deprecated/outdated:**

- `node12` runtime: Used by `publish-shell-action@v1`; GitHub runners still support it via FORCE override to node24, but node12 has been EOL since April 2022
- `node16` runtime: Already removed from GitHub runners
- `FORCE_JAVASCRIPT_ACTIONS_TO_NODE20` does not exist; the only FORCE flag is for node24

## Open Questions

1. **Will FORCE flag correctly override node12 -> node24 for publish-shell-action?**
   - What we know: The FORCE flag is documented to override `node20` to `node24`. It likely also handles `node12` and `node16` since the runner only ships node20 and node24 binaries.
   - What's unclear: Whether `publish-shell-action`'s code is compatible with node24 (it was written for node12).
   - Recommendation: Include `publish.yml` in the FORCE flag addition but note that this workflow only runs on push to main on the fork. If it fails, it is informational and does not block Phase 1. Document findings in baseline results.

2. **Should test-integration.yml also get the FORCE flag?**
   - What we know: `test-integration.yml` only calls a reusable workflow, it has no direct action `uses:` steps (only `uses: ./.github/workflows/...` which is a workflow call, not an action).
   - What's unclear: Whether the runner applies FORCE to the reusable workflow dispatch mechanism itself.
   - Recommendation: Add FORCE flag to `test-integration.yml` for completeness (it does no harm), but the critical placement is inside `integration-test-workflow.yml`.

## Validation Architecture

### Test Framework

| Property           | Value                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------ |
| Framework          | GitHub Actions CI (no local test framework)                                                            |
| Config file        | `.github/workflows/test.yml`, `.github/workflows/test-integration.yml`, `.github/workflows/format.yml` |
| Quick run command  | Push to branch and observe CI results                                                                  |
| Full suite command | Push to branch, wait for all 3 workflows to complete across all matrix entries                         |

### Phase Requirements to Test Map

| Req ID  | Behavior                                         | Test Type | Automated Command                          | File Exists?    |
| ------- | ------------------------------------------------ | --------- | ------------------------------------------ | --------------- |
| BVAL-02 | All existing CI tests pass on Node.js 24 runtime | e2e (CI)  | Push branch, check all workflow runs pass  | n/a -- CI-based |
| BVAL-03 | Validate action works end-to-end with FORCE flag | e2e (CI)  | Same as above; FORCE flag is the mechanism | n/a -- CI-based |

### Sampling Rate

- **Per task commit:** Push branch, check CI status
- **Per wave merge:** All 3 workflows green on all platforms
- **Phase gate:** Full matrix green + results documented in `01-BASELINE-RESULTS.md`

### Wave 0 Gaps

None -- this phase uses existing CI infrastructure. No new test files or frameworks needed.

## Sources

### Primary (HIGH confidence)

- GitHub Changelog: [Deprecation of Node 20 on GitHub Actions runners](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/) - FORCE flag documentation and timeline
- `gh api repos/actions/checkout/releases` - v5.0.0 (2025-08-11) and v6.0.0 (2025-11-20) release notes, confirmed node24 runtime
- `gh api repos/actions/checkout/contents/action.yml` - Confirmed `using: node24` on main branch
- `gh api repos/oven-sh/setup-bun/contents/action.yml` - Confirmed `using: "node24"` on main branch
- `gh api repos/JamesHenry/publish-shell-action/contents/action.yml` - Confirmed `using: node12`, only release is v1.0.0

### Secondary (MEDIUM confidence)

- [actions/runner#4295](https://github.com/actions/runner/issues/4295) - Deprecation warning fires incorrectly with FORCE flag (known bug)
- [GitHub community discussion #26671](https://github.com/orgs/community/discussions/26671) - Reusable workflows do not inherit caller env vars
- [actions/checkout#2286](https://github.com/actions/checkout/pull/2286) - v6 persist-credentials change details

### Tertiary (LOW confidence)

- None -- all findings verified with primary or secondary sources

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH - Versions verified directly via GitHub API against actual action.yml files
- Architecture: HIGH - FORCE flag behavior documented by GitHub; env isolation for reusable workflows is well-documented platform behavior
- Pitfalls: HIGH - Known bugs confirmed via official issue tracker; v6 credential changes confirmed via official PR

**Research date:** 2026-03-17
**Valid until:** 2026-06-01 (before GitHub's node24-default switchover date)
