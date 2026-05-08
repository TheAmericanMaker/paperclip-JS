# Closeout — 2026-05-07 — architecture

## Summary

Completed the architecture phase for the Paperclip monorepo. Produced `findings/architecture/architecture-map.md` with system intent, layer map (17 packages/modules), dependency direction, public surfaces (CLI, UI SPA, REST API, WebSocket, MCP, plugin bridge), runtime lifecycle (boot → reconcile → timers → shutdown), concurrency model (single-threaded Node.js + child-process offloading), build/packaging notes, and porting priorities. Validation passed all 5 criteria. 2 open questions and 7 carry-forward items recorded in `workflow/status.yaml`. Next phase ready: `defect-scan-mechanical`.

## Files Touched

- **Added:**
  - `.codecarto/findings/architecture/architecture-map.md`
  - `.codecarto/closeouts/2026-05-07-architecture.md`
- **Modified:**
  - `.codecarto/workflow/status.yaml`
  - `.codecarto/THREAD_LOG.md`
- **Deleted:**
  - (none)

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Phase validation per VALIDATE.md | PASS | All 5 criteria satisfied; appended to primary output. |
| Source read-only rule | PASS | No modifications outside `.codecarto/`. |

## Decisions Beyond Prompt

- **D001** | Used external dependency `hermes-paperclip-adapter` as a first-class adapter in the architecture map despite it not being a workspace package. | It is imported directly by the server adapter registry alongside workspace adapters, so it is architecturally part of the adapter layer even though it lives on npm.

## Proposed Conventions

### C1 Adapter export triplet pattern

**Why:** Every workspace adapter package (`claude-local`, `codex-local`, `cursor-local`, `gemini-local`, `opencode-local`, `pi-local`, `acpx-local`, `openclaw-gateway`) exports exactly three subpaths: `/server`, `/ui`, `/cli`. This is a project-wide invariant.

**How to apply:** When documenting or porting adapters, always assume the triplet exists. If an adapter deviates, flag it as an anomaly.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| arch-OQ1 | needs-runtime-test | Does the system use any in-memory caching (e.g., Redis, LRU) beyond DB? | Not visible from source alone; runtime test or production deployment inspection needed. |
| arch-OQ2 | needs-maintainer-decision | Are there plans to extract the heartbeat scheduler into a standalone worker process, or will it remain in the main server process? | README mentions "coming soon" cloud/sandbox agents; architecture may shift. |

## Carry-Forward Routed

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| arch-CF1 | protocols | WebSocket live event message schemas and sequence diagrams are not yet extracted. | Wire-format extraction is the protocols phase's rubric. |
| arch-CF2 | protocols | Plugin worker bridge protocol (JSON-RPC shapes, capability negotiation, host service method catalog) not yet fully documented. | Protocol schema extraction belongs to protocols phase. |
| arch-CF3 | contracts | Detailed per-route request/response validation schemas (Zod validators exist in shared but are not tied to user-facing contracts here). | Contracts phase maps user-visible behavior and acceptance checks. |
| arch-CF4 | contracts | Adapter execution context shape, session codec behavior, and runtime command spec contracts. | Contracts phase covers black-box acceptance for adapters. |
| arch-CF5 | defect-scan-mechanical | `cursor-local` adapter `getRuntimeCommandSpec` uses `buildCursorRuntimeCommandSpec` with `installCommand: null`; manual install assumption may be a config hazard. | Mechanical defect scan rubric. |
| arch-CF6 | defect-scan-mechanical | `hermes-paperclip-adapter` is imported as an external npm dep with an `as unknown` cast for `authToken`; version mismatch risk. | Mechanical defect scan rubric. |
| arch-CF7 | build-and-deploy | Full CI/CD matrix (Node versions, OS targets, sandbox-provider plugin packaging) is only partially read. | Secondary output `findings/build-and-deploy/build-and-deploy.md` will accumulate this. |

## Framework Feedback (optional)

No framework friction encountered. The pipeline YAML and SKILL.md instructions were clear and sufficient to complete the phase without ambiguity.

## Next Session Pointer

The next session should pick up `defect-scan-mechanical` (status is `pending`, `current_phase` in `workflow/status.yaml` is now `defect-scan-mechanical`). Required reads: `GUIDE.md`, `workflow/status.yaml`, `findings/architecture/architecture-map.md`, and `findings/defect-scan-mechanical/SKILL.md`. The carry-forward items `arch-CF5` and `arch-CF6` are already staged in `workflow/status.yaml` for that phase to resolve.
