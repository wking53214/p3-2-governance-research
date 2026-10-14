# Relational Schema: The Proposal, Its Caveats, and What the Prototype Actually Keeps

**Status:** The schema in `white-paper-v1.md` section 2 is a design. The prototype keeps a smaller set of state and found two of the schema's columns undefined or unsafe. This document is the reconciliation.

---

## The proposed tables, in one paragraph each

- **trap_events**: append-only, one row per enforcement action (which context, which boundary, outcome, recovery latency, metadata). The source of truth.
- **execution_contexts**: a cache of running call chains, hashed to keep cardinality down, with a parent link for nesting and a last-activity time for ageing.
- **boundaries**: small reference table of rules (type, layer, threshold, severity) plus `effectiveness_score`.
- **deadline_windows**: temporal constraints, created and closed per execution.
- **trap_patterns**: materialised summaries per (context hash, boundary): trap counts over 1h, 24h, 7d, mean recovery latency, `effectiveness_signal`, `digest_timestamp`. The live path reads this, not the events.

Hot, warm and cold tiers by age; partitioning by tenant and domain.

## Caveats found since the white paper

1. **`effectiveness_score` and `effectiveness_signal` were undefined.** The prototype now defines effectiveness for one boundary type, the deadline, in one currency: time per task. Benefit is the share of tasks trapped at the original deadline times the cut; cost is the share newly killed at the tightened deadline times that deadline. The verdict is categorical (IMPROVED, UNCHANGED, DEGRADED, UNKNOWN) on the confidence bounds of those two numbers. For token and memory caps no benefit was found to measure, so no score is defined. See `experiments/validation-framework.md`. The column should hold the last verdict and its two bounded numbers, not a 0 to 1 float.
2. **`SELECT MAX(digest_timestamp)` is not a safe checkpoint.** See `implementation/materialization-correctness.md`.
3. **trap_patterns needs two more counters** for validation: per (context, boundary, time bucket), how many tasks ran on the canary (original) limit and how many of those trapped. Without them the validation layer has nothing to compare.
4. **The boundary's current limit is the live read, not the trap counts.** The kernel reads `GET /boundaries/<agent>.<limit>` before an execution and obeys it. The pattern counts drive the governor's decision; the enforcement layer only needs the resulting limit. That is a one-row lookup by key and does not need the time-bucketed counts on the hot path.

## What the prototype keeps instead

| Proposed | Prototype | Why |
|---|---|---|
| trap_events table | Trap events arrive from the kernel in the P3.2 schema and are consumed; the durable record is the hash-chained audit log (`boundary_version`, `authority_decision`, `signal`, `tightening_reviewed` entries) | The audit log is what has to be tamper-evident and replayable; raw traps can be regenerated or re-streamed |
| execution_contexts | One boundary per agent per numeric limit (`stack.agent.<id>.deadline_ns`); agent ids sanitised and capped at 1,000 | Agents are the unit the kernel enforces on; no call-stack hashing was needed at this scale |
| trap_patterns | In-memory per-agent windows: last 50 trap ids, trap stamps over the last 2,000 tasks, per-slice counts (tasks, traps, canary tasks, canary traps), frozen baseline quantiles | Same information, scoped to what the rules read |
| boundaries | `BoundaryStore` with version history, grants, floor | Plus the invariant checks the schema cannot express |
| deadline_windows | The kernel's own transaction deadline | Not duplicated |

The prototype is in-process Python with a file-backed audit log. The core claim (the ratchet, the rules, the validation) does not depend on the database, which is why the database was not built. What a production store must add is in `implementation/technology-choices.md`.
