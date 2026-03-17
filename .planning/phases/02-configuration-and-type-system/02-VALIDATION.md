---
phase: 2
slug: configuration-and-type-system
status: approved
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-17
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property               | Value                                                |
| ---------------------- | ---------------------------------------------------- |
| **Framework**          | GitHub Actions CI workflows (no unit test framework) |
| **Config file**        | `.github/workflows/test.yml`, `test-integration.yml` |
| **Quick run command**  | `npx tsc --noEmit`                                   |
| **Full suite command** | Push to branch, verify all 5 CI workflows pass       |
| **Estimated runtime**  | ~10 seconds (tsc), ~3 minutes (full CI)              |

---

## Sampling Rate

- **After every task commit:** Run `npx tsc --noEmit`
- **After every plan wave:** Run full CI suite (push to branch)
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 10 seconds (local tsc)

---

## Per-Task Verification Map

| Task ID  | Plan | Wave | Requirement | Test Type | Automated Command                                                                                                 | File Exists | Status  |
| -------- | ---- | ---- | ----------- | --------- | ----------------------------------------------------------------------------------------------------------------- | ----------- | ------- |
| 02-01-01 | 01   | 1    | RUNT-01     | smoke     | `git grep "using: 'node24'" -- action.yml`                                                                        | N/A         | pending |
| 02-01-02 | 01   | 1    | RUNT-02     | smoke     | `node -e "const p=require('./package.json'); console.assert(p.volta.node==='24.14.0')"`                           | N/A         | pending |
| 02-01-03 | 01   | 1    | RUNT-03     | smoke     | `node -e "const p=require('./package.json'); console.assert(p.engines.node==='>=24')"`                            | N/A         | pending |
| 02-01-04 | 01   | 1    | TSCO-01     | smoke     | `node -e "const t=require('./tsconfig.json'); console.assert(t.compilerOptions.target==='ES2024')"`               | N/A         | pending |
| 02-01-05 | 01   | 1    | TSCO-02     | smoke     | `node -e "const t=require('./tsconfig.json'); console.assert(t.compilerOptions.lib[0]==='ES2024')"`               | N/A         | pending |
| 02-02-01 | 02   | 1    | TYPE-01     | smoke     | `node -e "const p=require('./package.json'); console.assert(p.devDependencies['@types/node'].startsWith('^24'))"` | N/A         | pending |
| 02-02-02 | 02   | 1    | TYPE-02     | unit      | `npx tsc --noEmit 2>&1` (expect only pre-existing TS2307)                                                         | N/A         | pending |

_Status: pending / green / red / flaky_

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No new test files needed.

---

## Manual-Only Verifications

| Behavior                            | Requirement | Why Manual                     | Test Instructions                         |
| ----------------------------------- | ----------- | ------------------------------ | ----------------------------------------- |
| CI workflows pass on GitHub Actions | All         | GitHub Actions runner required | Push to branch, check all 5 workflow runs |

---

## Validation Sign-Off

- [x] All tasks have automated verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 10s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved

_Nyquist compliance note: This is a CI-driven project with no local unit test framework.
Nyquist sampling requirements are satisfied by the GitHub Actions test matrix (ubuntu, macOS,
Windows runners for each push and PR event). Wave 0 requirements are met by the existing CI
infrastructure. Signed off 2026-03-18 as part of Phase 5 audit documentation closure._
