---
phase: 3
slug: source-fixes-and-build
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-17
---

# Phase 3 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property               | Value                                                  |
| ---------------------- | ------------------------------------------------------ |
| **Framework**          | GitHub Actions CI (integration tests via `uses: ./`)   |
| **Config file**        | `.github/workflows/test.yml`                           |
| **Quick run command**  | `npx tsc --noEmit && bun run build`                    |
| **Full suite command** | `git push` + CI green on test.yml (3-OS matrix)        |
| **Estimated runtime**  | ~120 seconds (CI matrix across ubuntu, macos, windows) |

---

## Sampling Rate

- **After every task commit:** Run `npx tsc --noEmit && bun run build`
- **After every plan wave:** Run full CI via `git push`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 120 seconds

---

## Per-Task Verification Map

| Task ID  | Plan | Wave | Requirement | Test Type   | Automated Command                   | File Exists       | Status  |
| -------- | ---- | ---- | ----------- | ----------- | ----------------------------------- | ----------------- | ------- |
| 03-01-01 | 01   | 1    | AUDT-01     | manual-only | N/A -- documentation deliverable    | N/A               | pending |
| 03-01-02 | 01   | 1    | AUDT-02     | manual-only | N/A -- documentation deliverable    | N/A               | pending |
| 03-01-03 | 01   | 1    | AUDT-03     | manual-only | N/A -- documentation deliverable    | N/A               | pending |
| 03-02-01 | 02   | 1    | BVAL-01     | smoke       | `bun run build && npx tsc --noEmit` | N/A (CI workflow) | pending |
| 03-02-02 | 02   | 1    | BVAL-01     | integration | `git push` + CI green               | N/A (CI workflow) | pending |

_Status: pending / green / red / flaky_

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements.

No unit test framework needed -- the project uses integration testing via CI workflows that run the action with `uses: ./`. The 3-OS matrix (ubuntu, macos, windows) in test.yml provides comprehensive coverage.

---

## Manual-Only Verifications

| Behavior                             | Requirement | Why Manual                          | Test Instructions                                                                                                      |
| ------------------------------------ | ----------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Node.js API audit for nx-set-shas.ts | AUDT-01     | Documentation deliverable, not code | Review 03-AUDIT.md for completeness: all Node.js built-in APIs listed with file/line, Node.js 24 status, action needed |
| Node.js API audit for pre-commit.ts  | AUDT-02     | Documentation deliverable, not code | Review 03-AUDIT.md for pre-commit.ts section: execSync, process.exit covered                                           |
| Audit findings documented            | AUDT-03     | Documentation deliverable, not code | Verify 03-AUDIT.md exists in phase directory with checklist table format                                               |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 120s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
