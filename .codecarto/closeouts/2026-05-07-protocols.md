# Closeout — 2026-05-07 — protocols

## Summary

This session closed the protocols phase by extracting wire formats, state machines, and persistence semantics from the Paperclip server and plugin SDK. It resolved four carry-forward items routed from earlier phases: WebSocket live event schemas (arch-CF1 / contracts-CF1) and the plugin worker bridge protocol (arch-CF2 / contracts-CF2). The output catalogs five protocol boundaries, three state machines (plugin worker lifecycle, heartbeat run lifecycle, plugin job lifecycle), persistent schema notes for six storage mechanisms, and ten portability hazards. The next session should pick up `defect-scan-semantic` per the updated `workflow/status.yaml`.

## Files Touched

- **Added:**
  - `.codecarto/findings/protocols/protocols-and-state.md`
  - `.codecarto/closeouts/2026-05-07-protocols.md`
- **Modified:**
  - `.codecarto/workflow/status.yaml`
  - `.codecarto/THREAD_LOG.md`

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation per workflow/VALIDATE.md | PASS | All 5 criteria satisfied; appended to primary output. |
| Source code modifications | N/A | No source files outside `.codecarto/` were modified. |

## Decisions Beyond Prompt

- None. All work followed the protocols phase SKILL.md and template without divergence.

## Proposed Conventions

- None new this session.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| protocols-OQ2 | needs-runtime-test | What is the actual max WebSocket payload size or backpressure policy enforced by the ws library configuration? | Not visible from source; ws defaults apply unless overridden. |
| protocols-OQ3 | needs-maintainer-decision | Are plugin worker environment variables intentionally minimal (no DATABASE_URL, no host secrets), or should future plugins receive additional host context? | Current code explicitly strips host env for security; policy decision. |

## Carry-Forward Routed

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| protocols-CF1 | defect-scan-semantic | True atomicity of checkout+budget enforcement — is it one transaction or two? (mirrored from contracts-CF3) | Semantic pass 3 (concurrency) will verify transaction boundaries. |
| protocols-CF2 | porting | Detailed UI component-level behavior and React Query cache policies. (mirrored from contracts-CF4) | Porting synthesis needs high-level feature parity, not component internals. |

## Framework Feedback (optional)

None.

## Next Session Pointer

The next phase is `defect-scan-semantic`. It depends on `architecture`, `contracts`, `protocols`, and `defect-scan-mechanical` — all now complete. The session should read `findings/defect-scan-semantic/SKILL.md`, the architecture map, contracts, this protocols output, and the mechanical defects report. It must resolve or re-route `protocols-CF1` (checkout+budget atomicity) as part of its concurrency pass.
