---
phase: 4
slug: version-bump-and-release-pr
status: approved
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-17
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property               | Value                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------ |
| **Framework**          | GitHub Actions CI + inline smoke checks                                                                |
| **Config file**        | `.github/workflows/test.yml`, `.github/workflows/test-integration.yml`, `.github/workflows/format.yml` |
| **Quick run command**  | `bun run format:check`                                                                                 |
| **Full suite command** | Push to `origin/feat/node24-runtime` and monitor GitHub Actions                                        |
| **Estimated runtime**  | ~3 minutes (CI) / ~5 seconds (local format check)                                                      |

---

## Sampling Rate

- **After every task commit:** Run `bun run format:check` + `git status --short -- dist/`
- **After every plan wave:** N/A (single wave)
- **Before `/gsd:verify-work`:** Full CI suite must be green on `origin/feat/node24-runtime`
- **Max feedback latency:** ~5 seconds (local checks); ~3 minutes (CI)

---

## Per-Task Verification Map

| Task ID  | Plan | Wave | Requirement | Test Type   | Automated Command                                                                                                   | File Exists                  | Status  |
| -------- | ---- | ---- | ----------- | ----------- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------- | ------- |
| 04-01-01 | 01   | 1    | RLSE-01     | smoke       | `node -e "const p=require('./package.json'); if(p.version!=='5.0.0') throw new Error('wrong version: '+p.version)"` | N/A (inline)                 | pending |
| 04-01-02 | 01   | 1    | RLSE-02     | smoke       | `git status --short -- dist/` (outputs nothing)                                                                     | N/A (git check)              | pending |
| 04-01-03 | 01   | 1    | RLSE-03     | integration | Push to origin; observe GitHub Actions (test.yml, test-integration.yml, format.yml)                                 | `.github/workflows/test.yml` | pending |
| 04-01-04 | 01   | 1    | RLSE-04     | manual      | `gh pr list --repo nrwl/nx-set-shas --head LayZeeDK:feat/node24-runtime`                                            | N/A (PR check)               | pending |

_Status: pending / green / red / flaky_

---

## Wave 0 Requirements

None — existing CI infrastructure covers all phase requirements. No new test files needed.

---

## Manual-Only Verifications

| Behavior                                                    | Requirement | Why Manual                                                       | Test Instructions                                                                                                              |
| ----------------------------------------------------------- | ----------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| PR exists against upstream with breaking changes documented | RLSE-04     | PR creation is a GitHub state operation, not automatable locally | Run `gh pr list --repo nrwl/nx-set-shas --head LayZeeDK:feat/node24-runtime` and confirm PR URL returned                       |
| Fork CI workflows pass (test, test-integration, format)     | RLSE-03     | `format.yml` only triggers on PR, not on push                    | Open GitHub Actions tab at `https://github.com/LayZeeDK/nrwl-nx-set-shas/actions` after push; verify all three workflows green |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 10s (local) / < 5m (CI)
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved

_Nyquist compliance note: This is a CI-driven project with no local unit test framework.
Nyquist sampling requirements are satisfied by the GitHub Actions test matrix (ubuntu, macOS,
Windows runners for each push and PR event). Wave 0 requirements are met by the existing CI
infrastructure. Signed off 2026-03-18 as part of Phase 5 audit documentation closure._
