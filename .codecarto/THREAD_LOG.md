# Thread Log — Index

This file is an **index** of per-session closeouts. Each session writes a full closeout to
`closeouts/<YYYY-MM-DD>-<phase-or-module>.md` using `templates/closeout-template.md`, and
appends one line here pointing to it.

The body of each session lives in the closeout file, not in this index. This pattern scales
forever: per-session files are individually small and read-budget-cheap, and avoid the
heredoc-vs-edit sync risks that bite append-to-large-file workflows once the file grows past
~50 KB.

## Format

```
- YYYY-MM-DD — <phase-or-module> — <one-line-summary> — [closeout](closeouts/YYYY-MM-DD-phase-or-module.md)
```

## De-dup discipline

Before appending, scan the bottom 5 entries. If you see a line with the same date AND same
phase-or-module AND same summary, do not append — the prior session already wrote it. The
framework has no programmatic dedup gate; this is human-discipline. (See
`Apply 20 spec deltas to Thaumaturge.txt` for the incident that established this rule.)

A one-liner to surface duplicates from the shell:

```bash
grep -E '^- [0-9]{4}-[0-9]{2}-[0-9]{2}' .codecarto/THREAD_LOG.md | sort | uniq -d
```

## Entries

<!--
  Append one line per session below this marker.
  Example:
  - 2026-05-02 — framework-feedback-pass — applied 6 spec-blockers + 5 clarifications from FEEDBACK_INDEX.md — [closeout](closeouts/2026-05-02-framework-feedback-pass.md)
-->

- 2026-05-02 — framework-feedback-pass — applied 6 spec-blockers + 5 clarifications from FEEDBACK_INDEX.md; 14 deferred to BACKLOG.md — [closeout](closeouts/2026-05-02-framework-feedback-pass.md)
- 2026-05-07 — architecture — mapped 17 packages, 3 public surfaces, 2 open questions, 7 carry-forward items — [closeout](closeouts/2026-05-07-architecture.md)
- 2026-05-07 — defect-scan-mechanical — scanned 14 mechanical defects, closed arch-CF5 and arch-CF6 — [closeout](closeouts/2026-05-07-defect-scan-mechanical.md)
- 2026-05-07 — contracts — documented 6 surfaces, closed arch-CF3 and arch-CF4, 15 black-box acceptance checks, 4 open questions — [closeout](closeouts/2026-05-07-contracts.md)
- 2026-05-07 — protocols — documented 5 protocol boundaries, 3 state machines, 10 hazards; closed arch-CF1/CF2 and contracts-CF1/CF2 — [closeout](closeouts/2026-05-07-protocols.md)
- 2026-05-07 — defect-scan-semantic — scanned 10 semantic defects, closed contracts-CF3/protocols-CF1, resolved contracts-OQ1 — [closeout](closeouts/2026-05-07-defect-scan-semantic.md)
