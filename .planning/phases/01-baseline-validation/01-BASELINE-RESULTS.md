# Baseline Validation Results

**Date:** 2026-03-17
**Branch:** `LayZeeDK/feat/migrate-to-node24-runtime`
**PR:** https://github.com/LayZeeDK/nrwl-nx-set-shas/pull/1

## Dependency Versions

| Dependency                        | Previous | Updated        | Notes                                  |
| --------------------------------- | -------- | -------------- | -------------------------------------- |
| `actions/checkout`                | v4       | v6             | node24-native                          |
| `oven-sh/setup-bun`               | v2       | v2 (unchanged) | Already node24-native                  |
| `jameshenry/publish-shell-action` | v1       | v1 (unchanged) | No newer version; note for upstream PR |

## Workflow x Platform Matrix

### Push-triggered

| Workflow         | ubuntu-latest | macOS-latest | Windows-latest | Run                                                                                  |
| ---------------- | :-----------: | :----------: | :------------: | ------------------------------------------------------------------------------------ |
| Test             |     PASS      |     PASS     |      PASS      | [23192254202](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23192254202) |
| Test Integration |     PASS      |     PASS     |      PASS      | [23192254217](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23192254217) |
| Publish          |     PASS      |      -       |       -        | [23192600890](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23192600890) |

### PR-triggered (PR #1)

| Workflow         | ubuntu-latest | macOS-latest | Windows-latest | Run                                                                                  |
| ---------------- | :-----------: | :----------: | :------------: | ------------------------------------------------------------------------------------ |
| Test             |     PASS      |     PASS     |      PASS      | [23192260192](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23192260192) |
| Test Integration |     PASS      |     PASS     |      PASS      | [23192260288](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23192260288) |
| Check Formatting |     PASS      |      -       |       -        | [23192602202](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23192602202) |

## Notes

### Publish workflow

Publish ran successfully on Node 24 (`FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true`) but was a no-op: "Tag v4.4.0 already exists". The `jameshenry/publish-shell-action@v1` is still on Node 20 (deprecation warning emitted) but functions correctly when forced to Node 24. No newer version exists.

### Check Formatting (initial run)

The first PR-triggered Check Formatting run ([23192260185](https://github.com/LayZeeDK/nrwl-nx-set-shas/actions/runs/23192260185)) failed due to unformatted `.planning/` markdown files (GSD artifacts). This was unrelated to Node 24. Resolved by applying Prettier formatting to all `.planning/` files via rebase. Subsequent run passes.

## NX_BASE / NX_HEAD Verification

Both push and PR workflows include verification steps that assert `NX_BASE` and `NX_HEAD` are set correctly:

- **PR workflow:** Verifies `NX_BASE == git merge-base origin/<base-ref> HEAD` and `NX_HEAD == HEAD`
- **Push workflow:** Verifies `NX_BASE` is an ancestor of HEAD and `NX_HEAD == HEAD`

All 12 platform x workflow x trigger combinations passed these assertions.

## Conclusion

**Result: The action works correctly on Node.js 24 runtime with zero source code changes.**

- All Test and Test Integration workflows pass on all three platforms (ubuntu, macOS, Windows)
- Both push and PR event triggers work correctly
- `NX_BASE` and `NX_HEAD` outputs are computed correctly
- Publish workflow runs successfully on Node 24 (no-op due to existing tag)
- Check Formatting passes after formatting fixes
- The `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` flag successfully forces Node.js 24 runtime

### Scope Impact on Phases 2-3

The baseline confirms HIGH confidence from the research phase. Since the action already works on Node.js 24:

- **Phase 2 (Source Compatibility):** May need minimal or no source changes. Focus shifts to updating `action.yml` to declare `node24` runtime natively.
- **Phase 3 (CI & Docs):** Remove the temporary `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` flag and finalize the `action.yml` node version declaration.
