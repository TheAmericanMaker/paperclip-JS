# Closeout — 2026-05-07 — defect-scan-mechanical

## Summary

Ran the mechanical defect-scan phase across the Paperclip repository. Scanned passes 1 (logic/correctness), 2 (error handling), and 6 (config/environment) on the server, adapters, and shared packages. Documented 14 findings: 1 high, 5 medium, 8 low. Closed two architecture carry-forward items: arch-CF5 (cursor adapter installCommand null hazard) and arch-CF6 (hermes-paperclip-adapter as-unknown cast risk). No items were routed to the semantic phase. The next session should begin the contracts phase, which can reference the mechanical-defects.md findings when drafting behavioral contracts.

## Files Touched

- **Added:**
  - `.codecarto/findings/defect-scan-mechanical/mechanical-defects.md`
- **Modified:**
  - `.codecarto/workflow/status.yaml`
  - `.codecarto/THREAD_LOG.md`
- **Deleted:**
  - (none)

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation per workflow/VALIDATE.md | PASS | All 6 criteria passed; appended to primary output. |
| Source code modification | N/A | No source files outside `.codecarto/` were modified. |

## Decisions Beyond Prompt

- **D001** | Consolidated multiple bare-catch defects into representative findings rather than listing every empty catch block individually. | The codebase has ~30 bare catch blocks; listing each as a separate finding would drown the report. Instead, 2.3 and 2.6 aggregate patterns by functional area (startup path, registry reads) while retaining specific file references.
- **D002** | Rated the hermes `as unknown` cast as Medium rather than High. | The cast is documented with an inline comment explaining the version gap, and the dependency is pinned to `^0.2.0` in package.json, which limits the blast radius. A High rating would require evidence of an actual runtime crash or security exposure.

## Proposed Conventions

### C1 Bare-catch logging discipline

**Why:** The codebase uses bare `catch { }` extensively for defensive file I/O. When these blocks silently swallow errors, operators cannot distinguish "file missing" from "file corrupted," making debugging expensive.

**How to apply:** Any `catch` that returns a default value after a file read or JSON parse should log at `warn` level with the error message and file path before returning the fallback.

### C2 Env-var numeric validation pattern

**Why:** `Number(process.env.FOO)` followed by `Math.max` or arithmetic is a recurring pattern that silently produces `NaN` on invalid input, which then propagates into scheduler math or config objects.

**How to apply:** Wrap env-var numeric reads in a helper that checks `Number.isFinite` and falls back to a documented default, logging a warning when the env var is present but non-numeric.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| (none) | — | — | — |

## Carry-Forward Routed

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| (none) | — | — | — |

## Framework Feedback (optional)

The `grep` tool on this Windows environment failed when searching from `../` (returned `Incorrect function` on `nul` paths). Sessions should prefer absolute paths (e.g., `C:/Users/.../paperclip-JS/server/src`) or paths relative to the repo root rather than `../` when using `grep`/`find` on Windows hosts.

## Next Session Pointer

The next session should run the **contracts** phase (`findings/contracts/SKILL.md`). It should read `findings/defect-scan-mechanical/mechanical-defects.md` first — several mechanical defects (e.g., 6.1 cursor install path, 6.2 hermes type cast, 2.1 config-file swallowing) describe user-visible failure modes that should be reflected in the behavioral contracts. The contracts phase also needs to resolve carry-forward items arch-CF3 and arch-CF4 from the architecture phase.
