---
phase: 1
slug: baseline-validation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-17
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property               | Value                                                                                                           |
| ---------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Framework**          | GitHub Actions CI (no local test framework)                                                                     |
| **Config file**        | `.github/workflows/test.yml`, `.github/workflows/integration-test-workflow.yml`, `.github/workflows/format.yml` |
| **Quick run command**  | `git push` (triggers CI on branch)                                                                              |
| **Full suite command** | Push branch, wait for all 3 workflows to complete across all matrix entries                                     |
| **Estimated runtime**  | ~5-10 minutes (CI pipeline)                                                                                     |

---

## Sampling Rate

- **After every task commit:** `git push` and check CI status
- **After every plan wave:** All 3 workflows green on all platforms (ubuntu, macOS, Windows)
- **Before `/gsd:verify-work`:** Full matrix green + results documented in `01-BASELINE-RESULTS.md`
- **Max feedback latency:** ~10 minutes (CI pipeline)

---

## Per-Task Verification Map

| Task ID  | Plan | Wave | Requirement      | Test Type | Automated Command                         | File Exists     | Status  |
| -------- | ---- | ---- | ---------------- | --------- | ----------------------------------------- | --------------- | ------- |
| 01-01-01 | 01   | 1    | BVAL-02, BVAL-03 | e2e (CI)  | Push branch, check all workflow runs pass | n/a -- CI-based | pending |

_Status: pending / green / red / flaky_

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No new test files or frameworks needed.

- CI workflows already exist and test the action
- Cross-platform matrix (ubuntu, macOS, Windows) already configured
- No local test framework needed -- all validation is CI-based

---

## Manual-Only Verifications

| Behavior                                            | Requirement | Why Manual                                                                                    | Test Instructions                                                                                                                              |
| --------------------------------------------------- | ----------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Verify FORCE flag is working (node24 actually used) | BVAL-03     | Runner logs need manual inspection; deprecation warnings are misleading (actions/runner#4295) | 1. Push branch 2. Open CI run details 3. Check runner verbose output for node24 binary path OR enable `ACTIONS_RUNNER_DEBUG: true` for one run |
| Record baseline results                             | BVAL-02     | Requires human analysis of CI results to fill in results document                             | 1. Wait for all workflows to complete 2. Create `01-BASELINE-RESULTS.md` with matrix table, CI links, failure details                          |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 600s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
