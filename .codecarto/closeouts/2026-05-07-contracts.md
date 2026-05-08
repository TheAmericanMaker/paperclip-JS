# Closeout — 2026-05-07 — contracts

## Summary

This session executed the contracts phase of the CodeCartographer pipeline for paperclip-JS. It produced the behavioral contracts document covering six user-facing surfaces (CLI, Web UI, REST API, WebSocket, MCP server, Plugin SDK), closed two architecture carry-forward items (arch-CF3 per-route Zod schemas, arch-CF4 adapter execution context/session codec/runtime command spec), and produced a 15-item black-box acceptance list. Four open questions remain (budget hard-stop mid-run behavior, routine default policies, secret rotation UX, WebSocket delivery guarantees). Four items were carried forward to protocols and defect-scan-semantic.

## Files Touched

- **Added:**
  - `.codecarto/findings/contracts/behavioral-contracts.md`
  - `.codecarto/closeouts/2026-05-07-contracts.md`
- **Modified:**
  - `.codecarto/workflow/status.yaml`
  - `.codecarto/THREAD_LOG.md`
- **Deleted:**
  - (none)

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation per workflow/VALIDATE.md | PASS | All 6 criteria passed; appended to primary output. |
| Source file modification | N/A | No source files outside `.codecarto/` were modified. |

## Decisions Beyond Prompt

- **D001** | Per-route validation schemas documented in contracts output rather than extracted to a secondary file | The SKILL.md suggested secondary outputs only when a phase grows too large. The schema inventory is large but fits naturally under the "arch-CF3 closure" section within the primary output, keeping the contract-to-schema mapping visible for downstream porting.
- **D002** | Cursor adapter `installCommand: null` documented as a contract hazard rather than hidden | The mechanical defect (6.1) was already logged, but the contracts phase explicitly surfaces it in the adapter runtime command spec table so porting teams know Cursor cannot self-install.

## Proposed Conventions

### C1 Adapter Contract Table Format

**Why:** Adapters share a common interface (`ServerAdapterModule`) but each has distinct capability flags, install behavior, and session management. A uniform table makes comparisons possible.

**How to apply:** When documenting adapter contracts, always include: `type`, `command`, `detectCommand`, `installCommand`, `supportsLocalAgentJwt`, `supportsInstructionsBundle`, `requiresMaterializedRuntimeSkills`, `sessionCodec`, `sessionManagement`.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| contracts-OQ1 | needs-runtime-test | Does the budget hard stop kill an in-flight heartbeat, or only block future ones? | Code shows gating before spawn, but unclear if mid-run budget check exists. |
| contracts-OQ2 | needs-runtime-test | What is the exact default concurrency and catch-up policy for routines? | Schema names suggest defaults, but runtime behavior needs fixture capture. |
| contracts-OQ3 | needs-maintainer-decision | Are there plans for automatic secret key rotation UX? | Not visible in source; current system is manual. |
| contracts-OQ4 | needs-runtime-test | Do WebSocket live events guarantee delivery, or are they best-effort fire-and-forget? | Need capture against running service. |

## Carry-Forward Routed

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| contracts-CF1 | protocols | WebSocket live event message schemas and sequence diagrams | Wire-format extraction is the protocols phase's rubric. |
| contracts-CF2 | protocols | Plugin worker bridge protocol (JSON-RPC shapes, capability negotiation, host service method catalog) | Protocol schema extraction belongs to protocols phase. |
| contracts-CF3 | defect-scan-semantic | True atomicity of checkout+budget enforcement — is it one transaction or two? | Semantic pass 3 (concurrency) will verify transaction boundaries. |
| contracts-CF4 | porting | Detailed UI component-level behavior and React Query cache policies | Porting synthesis needs high-level feature parity, not component internals. |

## Next Session Pointer

The next phase is **protocols**. Start by reading `findings/protocols/SKILL.md` and `templates/protocols-and-state.md`. Pick up the two carry-forward items from architecture and contracts that target protocols: **arch-CF1** (WebSocket live event message schemas and sequence diagrams) and **arch-CF2** (plugin worker bridge protocol). Also address **contracts-CF1** and **contracts-CF2** (same topics, reinforced from contracts phase). The architecture map already identifies the relevant source files: `server/src/realtime/live-events-ws.ts` and `server/src/services/plugin-worker-manager.ts` / `server/src/services/plugin-host-services.ts`. Focus on event catalogs, state machines, persistence formats, and compatibility hazards per the protocols skill.
