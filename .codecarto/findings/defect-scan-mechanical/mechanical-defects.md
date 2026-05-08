# Mechanical Defects Report — paperclip-JS

<!--
  Output for the `defect-scan-mechanical` phase.
  Covers passes 1, 2, and 6 from the defect-scan methodology.
  Semantic passes (3 concurrency, 4 security, 5 contract violations)
  deferred to `defect-scan-semantic` after protocols.
-->

## Scan Context

- **Source:** `../` (repository root)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Pipeline:** `workflow/pipeline-full-with-deep-audit.yaml`
- **Date:** 2026-05-07
- **Scope:** Mechanical passes only (1 logic, 2 error handling, 6 configuration).
  Semantic passes deferred to `defect-scan-semantic` after protocols.

---

## Pass 1: Logic and Correctness

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 1.1 | `server/src/config.ts:253,259` | `Number(process.env.PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES)` and `Number(process.env.PAPERCLIP_DB_BACKUP_RETENTION_DAYS)` accept non-numeric strings, producing `NaN`. `Math.max(1, NaN)` returns `NaN`, which propagates into `databaseBackupIntervalMinutes` / `databaseBackupRetentionDays` and can break scheduled backup arithmetic. | Medium | observed fact | fix before porting — validate with `Number.isFinite` before `Math.max`, or fall back to defaults on NaN. |
| 1.2 | `server/src/config.ts:294` | `Number(process.env.PORT) || fileConfig?.server.port || 3100` treats `PORT="0"` as falsy because `Number("0") === 0`, so the server falls back to 3100 instead of binding to an OS-assigned port. | Low | observed fact | port differently — use explicit `isNaN` / `=== undefined` check instead of `||` chaining for port. |
| 1.3 | `server/src/routes/adapters.ts:236-240` | Version suffix parser uses `packageName.match(/@(\d+\.\d+\.\d+.*)$/)` with greedy `.*` and `lastIndexOf("@")`. A malformed scoped package name like `@scope/name@1.2.3@extra` mis-parses the version as `1.2.3@extra` instead of rejecting it. | Low | strong inference | fix before porting — tighten regex to `[^@]*$` or validate extracted version with semver regex. |

---

## Pass 2: Error Handling and Resilience

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 2.1 | `server/src/config-file.ts:11-13` | `readConfigFile` swallows *all* JSON parse and Zod validation errors in a bare `catch { return null }`. A malformed `config.json` silently yields null, causing the server to boot with defaults and masking configuration drift. | Medium | observed fact | fix before porting — log the parse/validation error before returning null so operators know their config is being ignored. |
| 2.2 | `server/src/services/adapter-plugin-store.ts:86-91` and `:109-114` | Store and settings reads use bare `catch` blocks around `JSON.parse`. Corrupted `adapter-plugins.json` or `adapter-settings.json` silently reset to empty arrays, hiding on-disk corruption from administrators. | Medium | observed fact | fix before porting — log a warning when the store file exists but cannot be parsed. |
| 2.3 | `server/src/index.ts` (multiple) | Startup path contains several bare `catch` blocks: Tailscale bind detection (`:106`), embedded-Postgres PID file read (`:340`), existing-server reachability probe (`:353`), and port rewrite (`:375`). Each silently suppresses a distinct failure mode (missing `tailscale` binary, stale PID file, unreachable DB, malformed URL). | Low | observed fact | port differently — log warnings in each catch so startup diagnostics are visible. |
| 2.4 | `server/src/services/feedback.ts:348` | `parseRunLogEntries` loops over raw log lines and calls `JSON.parse(line)` with a bare `catch`. A single malformed line is dropped silently; if the log format changes, the entire trace bundle may be silently empty with no observability. | Low | observed fact | port differently — count and log malformed lines so format drift is detectable. |
| 2.5 | `server/src/adapters/cursor-models.ts:85` | `parseCursorModelsOutput` silently ignores JSON parse errors and falls through to plain-text heuristics. If the Cursor CLI changes its `agent models` output to a new JSON shape, the fallback text parser may produce a stale or empty model list without warning. | Low | observed fact | port differently — log when JSON parsing fails so adapter output format changes are observable. |
| 2.6 | `server/src/services/local-service-supervisor.ts:112` | `safeReadRegistryRecord` returns `null` on any JSON parse or I/O error, masking registry corruption. A stale or malformed runtime-service registry file causes the service to be treated as missing, potentially spawning duplicate processes. | Low | observed fact | port differently — log a warning when a registry file exists but fails to parse. |

---

## Pass 6: Configuration and Environment Hazards

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 6.1 | `server/src/adapters/registry.ts:153-161` (arch-CF5) | `buildCursorRuntimeCommandSpec` hardcodes `installCommand: null`. Cursor CLI is not distributed via npm; the official install path is a curl\|bash script. When the `agent` binary is missing from PATH, the server produces a runtime spawn failure with no pre-flight validation or clear user-facing error. | High | observed fact | fix before porting — add a `detectCommand` check or `testEnvironment` path that surfaces a clear "Cursor CLI not installed" error before attempting execution. |
| 6.2 | `server/src/adapters/registry.ts:113,117,376` (arch-CF6) | `hermes-paperclip-adapter` is imported as an external npm dependency (`^0.2.0`) and its `execute` function is cast with `as unknown as ServerAdapterModule["execute"]` because v0.2.0 predates the `authToken` field. The cast bypasses compile-time type safety; a future v0.3.0 with breaking execute-signature changes could cause runtime failures that TypeScript would otherwise catch. | Medium | observed fact | fix before porting — upgrade hermes-paperclip-adapter to a version whose `execute` type matches `ServerAdapterModule["execute"]`, or add a runtime shape-validation wrapper. |
| 6.3 | `server/src/config.ts:253,259` | `PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES` and `PAPERCLIP_DB_BACKUP_RETENTION_DAYS` env vars are converted with `Number()` and then `Math.max(1, …)`. Non-numeric values yield `NaN`, and `Math.max(1, NaN)` returns `NaN`, which pollutes the config object and may crash backup-scheduler arithmetic. | Medium | observed fact | fix before porting — guard with `Number.isFinite` and fall back to defaults on invalid input. |
| 6.4 | `server/src/routes/adapters.ts:256,421` | `npm install` and `npm uninstall` timeouts are hardcoded to `120_000` ms and `60_000` ms. No env-var or config override exists for slow networks or large adapter packages. | Low | observed fact | port differently — make timeouts configurable or derive them from a shared network-timeout constant. |
| 6.5 | `server/src/app.ts:68` | `express.json({ limit: "10mb" })` is hardcoded. Company import/export payloads can inline full portable packages; a large payload hits this limit with no admin-configurable override. | Low | observed fact | port differently — expose `PAPERCLIP_API_JSON_LIMIT_MB` or read from config file. |

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 1 |
| Medium | 5 |
| Low | 8 |
| **Total** | 14 |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 1. Logic and correctness | 0 | 0 | 1 | 2 | 3 |
| 2. Error handling | 0 | 0 | 2 | 4 | 6 |
| 6. Config and environment | 0 | 1 | 2 | 2 | 5 |
| **Total** | 0 | 1 | 5 | 8 | 14 |

### Top Findings

1. **6.1** `server/src/adapters/registry.ts` — Cursor adapter has no install path (`installCommand: null`), leading to opaque runtime failures when Cursor CLI is missing. **High** — fix before porting.
2. **2.1** `server/src/config-file.ts` — Malformed `config.json` is swallowed silently; server boots with defaults. **Medium** — fix before porting.
3. **6.3** `server/src/config.ts` — Non-numeric backup interval/retention env vars produce `NaN` via `Math.max`. **Medium** — fix before porting.
4. **2.2** `server/src/services/adapter-plugin-store.ts` — Corrupted adapter-plugin store JSON silently resets to empty. **Medium** — fix before porting.
5. **6.2** `server/src/adapters/registry.ts` — `hermes-paperclip-adapter` imported with `as unknown` cast; version-mismatch risk. **Medium** — fix before porting.

### Routed To Semantic Phase

No items were spotted during this phase that require concurrency, security, or API-contract context. The following carry-forward items from the architecture phase were closed in this report:

| ID | Description | Closed In |
|----|-------------|-----------|
| arch-CF5 | `cursor-local` adapter `getRuntimeCommandSpec` uses `buildCursorRuntimeCommandSpec` with `installCommand: null`; manual install assumption is a config hazard. | Pass 6, finding 6.1 |
| arch-CF6 | `hermes-paperclip-adapter` is imported as an external npm dep with an `as unknown` cast for `authToken`; version mismatch risk. | Pass 6, finding 6.2 |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | At least two of the three mechanical passes (1, 2, 6) produced findings or documented "no defects found." | PASS | Pass 1 has 3 findings, Pass 2 has 6, Pass 6 has 5. |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | All 14 findings include location, severity, evidence level, and action columns. |
| 3 | Findings are organized by pass and sorted by severity. | PASS | Findings grouped under Pass 1, Pass 2, Pass 6; within each pass sorted critical→high→medium→low. |
| 4 | Summary tables are complete and counts match the detailed findings. | PASS | Severity counts (high=1, medium=5, low=8, total=14) match detailed tables. Pass totals (3+6+5=14) reconcile. |
| 5 | Findings are marked with evidence levels. | PASS | Every finding uses `observed fact` or `strong inference`. |
| 6 | Items spotted that are actually semantic in nature are routed to defect-scan-semantic via carry_forward in workflow/status.yaml. | PASS | No semantic items identified; none routed. Architecture carry-forward items arch-CF5 and arch-CF6 were closed in this report. |

**Validated by:** 2026-05-07 (defect-scan-mechanical phase, session 1)
**Overall:** PASS
