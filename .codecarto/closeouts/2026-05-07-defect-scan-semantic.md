# Closeout — 2026-05-07 — defect-scan-semantic

## Summary

Ran the deep semantic defect scan (passes 3, 4, 5) for paperclip-JS after contracts and protocols were complete. Identified 10 semantic defects: 4 concurrency/resource issues, 3 security/trust-boundary gaps, and 3 API contract / state-machine drifts. Closed the two carry-forward items routed from earlier phases (contracts-CF3 and protocols-CF1) by confirming that budget check and checkout are **not** atomic — they are two separate DB interactions with a TOCTOU window. Also resolved contracts-OQ1 from source: budget hard stops **do** kill in-flight runs via `evaluateCostEvent` → `cancelActiveForAgentInternal`. Next session is `porting`, which should synthesize architecture, contracts, protocols, and both defect passes into the reverse-engineering bundle.

## Files Touched

- **Added:**
  - `.codecarto/findings/defect-scan-semantic/semantic-defects.md`
  - `.codecarto/closeouts/2026-05-07-defect-scan-semantic.md`
- **Modified:**
  - `.codecarto/workflow/status.yaml`
  - `.codecarto/THREAD_LOG.md`
- **Deleted:**
  - (none)

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation per `workflow/VALIDATE.md` | PASS | All 6 criteria passed; output appended to primary artifact. |
| Source read coverage | PARTIAL | Key files read: `heartbeat.ts`, `issues.ts`, `budgets.ts`, `costs.ts`, `plugin-worker-manager.ts`, `plugin-job-scheduler.ts`, `auth.ts`, `live-events-ws.ts`, `authz.ts`. Did not exhaustively read every route file; focused on high-trust and high-concurrency paths. |

## Decisions Beyond Prompt

- **D001** | Resolved contracts-OQ1 in status.yaml by deleting it | Source analysis of `budgets.ts` and `heartbeat.ts` showed that `evaluateCostEvent` calls `pauseAndCancelScopeForBudget` → `cancelBudgetScopeWork` → `cancelActiveForAgentInternal`, which SIGTERM/SIGKILLs active runs. The open question was framed as needing a runtime test, but the code path is explicit enough to answer from static analysis. |

## Proposed Conventions

### C1 "Evidence-level discipline for contract-drift findings"

**Why:** When a semantic pass finds that code diverges from a documented state machine or contract, the finding is stronger when it cites both the spec reference (protocols/contracts file + section) and the exact code location.

**How to apply:** In `defect-scan-semantic` output, require the "Spec Reference" column for every Pass 5 finding. Use section anchors like `protocols-and-state.md §Plugin Worker Lifecycle` so downstream phases can trace the chain.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| | | | |

## Carry-Forward Routed

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| | | | |

## Next Session Pointer

The next session is `porting`. It should read:
1. `findings/porting/SKILL.md`
2. `findings/porting/reverse-engineering-bundle.md` (if existing)
3. `templates/reverse-engineering-bundle.md`
4. All prior phase outputs: `architecture-map.md`, `behavioral-contracts.md`, `protocols-and-state.md`, `mechanical-defects.md`, `semantic-defects.md`

Key inputs for the porting phase:
- **Defect synthesis**: Consolidate 14 mechanical + 10 semantic defects into a porting-oriented view (fix before porting / port differently / leave behind).
- **Carry-forward remaining**: `contracts-CF4` and `protocols-CF2` (UI component-level behavior / React Query cache policies) are still deferred to porting.
- **Resolved open question**: contracts-OQ1 answer (budget hard stop kills in-flight runs) should be noted in the porting bundle’s behavior notes.
