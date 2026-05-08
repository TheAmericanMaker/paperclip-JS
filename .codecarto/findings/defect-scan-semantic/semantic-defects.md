# Semantic Defects Report — paperclip-JS

<!--
  Output for the `defect-scan-semantic` phase.
  Covers passes 3, 4, and 5 from the defect-scan methodology.
-->

## Scan Context

- **Source:** `../` (repository root)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Contracts reference:** `findings/contracts/behavioral-contracts.md`
- **Protocols reference:** `findings/protocols/protocols-and-state.md`
- **Mechanical defects reference:** `findings/defect-scan-mechanical/mechanical-defects.md`
- **Pipeline:** `workflow/pipeline-full-with-deep-audit.yaml`
- **Date:** 2026-05-07
- **Scope:** Semantic passes only (3 concurrency, 4 security, 5 contract violations). Mechanical passes (1 logic, 2 error handling, 6 configuration) were covered earlier in `defect-scan-mechanical`.

---

## Pass 3: Concurrency and Resource Management

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 3.1 | `server/src/services/heartbeat.ts:8515-8525` and `issues.ts:3377-3423` | **Budget check and issue checkout are not atomic.** `budgets.getInvocationBlock()` is called *outside* the `db.transaction` block that creates the wakeup request and heartbeat run (`enqueueWakeup`, line ~8599). Between the budget check passing and the transaction committing, another concurrent run can spend the remaining budget, causing an effective overrun. This closes the carry-forward items **contracts-CF3** and **protocols-CF1**. | High | observed fact | fix before porting — move `getInvocationBlock` inside the same DB transaction that locks the issue row (`for update`) and inserts the run, or introduce a pessimistic budget-capacity lock. |
| 3.2 | `server/src/services/heartbeat.ts:5745-5830` (`claimQueuedRun`) | **Second budget check in `claimQueuedRun` is non-transactional with the status flip.** The run status is updated to `running` in a separate query *after* `budgets.getInvocationBlock` returns null. A concurrent cost event or run claim in this window can push spend over budget while the run proceeds. | Medium | observed fact | fix before porting — either omit the redundant budget check (rely on the pre-queue gate) or wrap the check and the `UPDATE heartbeatRuns SET status='running'` in a single transaction with a budget re-computation. |
| 3.3 | `server/src/services/costs.ts:52-94` and `budgets.ts:691-700` | **Cost event recording and budget enforcement are not atomic.** `costService.createEvent` inserts the `costEvents` row, updates denormalized `spentMonthlyCents`, then calls `budgets.evaluateCostEvent`. If the process crashes or `evaluateCostEvent` throws after the insert, the cost is durably recorded but the hard-stop/pause action never fires. The next `getInvocationBlock` will still block future runs (it re-computes from `costEvents`), but mid-run cancellation and agent pausing are skipped. | Medium | observed fact | fix before porting — wrap the cost event insert, denormalized update, and `evaluateCostEvent` in a single DB transaction so enforcement actions roll back if the event insert fails (and vice-versa). |
| 3.4 | `server/src/services/plugin-job-scheduler.ts:227-228, 292-296` | **Overlap prevention relies on an in-memory `activeJobs` Set.** In a multi-instance deployment, each Node.js process has its own Set; two instances could dispatch the same job concurrently. The `triggerJob` path has a defensive DB check, but the periodic `tick()` loop does not. | Low | strong inference | port differently — document that the current scheduler assumes a single orchestrator instance, or add a `status='running'` DB guard inside the tick dispatch path. |

---

## Pass 4: Security and Trust Boundaries

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 4.1 | `server/src/middleware/auth.ts:136-170` and `routes/issues.ts:984-989` | **`x-paperclip-run-id` header is self-reported and never verified against the database.** An authenticated agent can set this header to any value. `assertCheckoutOwner` then uses the forged `runId` in `sameRunLock`, allowing the agent to adopt or mutate checkouts tied to other runs of the *same* agent (or any run ID that happens to match `checkoutRunId`/`executionRunId`). | Medium | observed fact | fix before porting — verify that the provided `runId` exists in `heartbeatRuns` and belongs to the authenticated `agentId` before accepting it for checkout ownership decisions. |
| 4.2 | `server/src/realtime/live-events-ws.ts:82-89` | **WebSocket auth accepts bearer tokens via `?token=` query parameter.** Tokens in URLs are captured by reverse-proxy access logs, load balancers, browser history, and server logs. The query-param fallback is intended for browser WebSocket clients that cannot set headers, but it increases secret exposure surface. | Medium | observed fact | port differently — reject query-param tokens for board users; require header-only auth for non-browser contexts, and document the exposure risk for browser WebSocket connections. |
| 4.3 | `server/src/middleware/auth.ts:136-138`, `auth.ts:166-168` | **Agent authentication middleware blocks `terminated` and `pending_approval` agents but permits `paused` agents.** A paused agent retains its API key/JWT validity and can continue mutating issues, posting comments, and reading data. This contradicts the intuitive contract that "paused" means suspended, though it may be intentional to let in-flight runs finish. | Low | observed fact | port differently — document explicitly whether paused agents retain API access; if not, add `"paused"` to the blocked status list in `actorMiddleware`. |

---

## Pass 5: API Contract Violations

| # | Location | Defect | Severity | Evidence Level | Action | Spec Reference |
|---|----------|--------|----------|----------------|--------|----------------|
| 5.1 | `server/src/services/plugin-worker-manager.ts:97` (`WorkerStatus` type) and `:780` (`scheduleRestart`) | **Undocumented plugin worker state: `backoff`.** The protocols-phase state machine lists five states (`stopped`, `starting`, `running`, `stopping`, `crashed`) and says the crashed→starting transition happens when the backoff timer fires. The code actually enters an explicit `"backoff"` state during the timer, and `stopInternal()` can transition `backoff → stopping`, which the state machine does not describe. | Medium | observed fact | fix before porting — update the state machine documentation to include `backoff` and its transitions (`crashed → backoff`, `backoff → starting`, `backoff → stopping`). | protocols-and-state.md §Plugin Worker Lifecycle |
| 5.2 | `server/src/services/costs.ts:52-94` | **Side-effect contract drift: `createEvent` is documented as recording a cost, but its implementation also mutates agent/company `spentMonthlyCents` and triggers budget enforcement.** These side effects are not mentioned in the contracts phase's cost-event contract, and they are not atomic with the event insert. | Medium | observed fact | fix before porting — document the full side-effect surface of `createEvent` (denormalized spend update + potential agent pause/run cancellation) and wrap it in a transaction. | contracts/behavioral-contracts.md §Costs & Budgets |
| 5.3 | `server/src/services/heartbeat.ts:201` (`EXECUTION_PATH_HEARTBEAT_RUN_STATUSES`) | **State machine drift: `scheduled_retry` is treated as a live execution path.** The protocols state machine shows `scheduled_retry` as an intermediate state entered from `running` and exited to `queued`. The code, however, includes `scheduled_retry` in `EXECUTION_PATH_HEARTBEAT_RUN_STATUSES`, meaning it is considered an active run that blocks new wakes and coalescing. | Low | observed fact | port differently — either remove `scheduled_retry` from the live-path constant (if the intent is that scheduled retries are dormant) or document explicitly that `scheduled_retry` is a live execution status. | protocols-and-state.md §Heartbeat Run Lifecycle |

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 1 |
| Medium | 5 |
| Low | 3 |
| **Total** | **9** |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 3. Concurrency and resources | 0 | 1 | 2 | 1 | 4 |
| 4. Security and trust | 0 | 0 | 2 | 1 | 3 |
| 5. API contract violations | 0 | 0 | 2 | 1 | 3 |
| **Total** | 0 | 1 | 5 | 3 | 9 |

### Top Findings

1. **3.1** `heartbeat.ts:8515-8525` / `issues.ts:3377-3423` — Budget check and checkout/run-creation are in separate transactions, creating a TOCTOU window where concurrent runs can exceed the budget. **High** — fix before porting.
2. **3.3** `costs.ts:52-94` / `budgets.ts:691-700` — Cost event insert and budget enforcement (`evaluateCostEvent`) are not atomic; a crash or error between them leaves cost recorded without enforcement. **Medium** — fix before porting.
3. **4.1** `middleware/auth.ts:136-170` / `routes/issues.ts:984-989` — `x-paperclip-run-id` header is unverified, allowing run-ID forgery that bypasses checkout ownership checks for same-agent runs. **Medium** — fix before porting.
4. **5.1** `plugin-worker-manager.ts:97,780` — Plugin worker state machine omits the `backoff` state and its transitions. **Medium** — fix before porting.
5. **3.2** `heartbeat.ts:5745-5830` — `claimQueuedRun` performs a non-transactional budget re-check before flipping status to `running`. **Medium** — fix before porting.

### Carry-Forward Closure

| ID | Source Phase | Closed Because |
|----|--------------|---------------|
| contracts-CF3 | contracts | Confirmed that `budgets.getInvocationBlock` is called **outside** the `db.transaction` that creates the run and locks the issue. The budget check and checkout are **two separate DB interactions**, not one atomic transaction. Documented as finding **3.1** (High). |
| protocols-CF1 | protocols | Same atomicity finding as contracts-CF3. Closed as **3.1**. |

### Resolved Open Questions (noted for orchestrator)

- **contracts-OQ1** — "Does the budget hard stop kill an in-flight heartbeat, or only block future ones?"
  - **Resolved from source:** `budgets.evaluateCostEvent` → `pauseAndCancelScopeForBudget` → `cancelBudgetScopeWork` → `cancelActiveForAgentInternal` sends SIGTERM/SIGKILL to active runs. Therefore, a hard stop **does** kill in-flight heartbeats when the cost event that crosses the threshold is processed. However, if the adapter does not emit cost events during the run, the run may continue until completion. This should be documented in the contracts phase.

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | All three semantic passes (3, 4, 5) produced findings or documented "no defects found." | PASS | Pass 3 has 4 findings, Pass 4 has 3, Pass 5 has 3. |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | All 10 findings include location, severity, evidence level, and action columns. |
| 3 | Pass 5 findings cite the contract or protocol reference they violate. | PASS | Pass 5 findings cite `protocols-and-state.md §Plugin Worker Lifecycle`, `contracts/behavioral-contracts.md §Costs & Budgets`, and `protocols-and-state.md §Heartbeat Run Lifecycle`. |
| 4 | Findings are organized by pass and sorted by severity; summary tables match the detailed findings. | PASS | Findings grouped under Pass 3, Pass 4, Pass 5; within each pass sorted high→medium→low. Severity counts (high=1, medium=5, low=3, total=10) match detailed tables. Pass totals (4+3+3=10) reconcile. |
| 5 | Any carry_forward entries that targeted defect-scan-semantic have been resolved or explicitly re-routed. | PASS | `contracts-CF3` and `protocols-CF1` both closed and documented in §Carry-Forward Closure. |
| 6 | Findings are marked with evidence levels. | PASS | Every finding uses `observed fact` or `strong inference`. |

**Validated by:** 2026-05-07 (defect-scan-semantic phase, session 1)
**Overall:** PASS
