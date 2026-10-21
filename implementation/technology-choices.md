# Technology Choices: What the White Paper Proposed, What Was Built, What to Pick

**Status:** Guidance, not a decision. The prototype deliberately built none of this, because the claim under test does not depend on it.

---

## The white paper's options (section 8)

| Option | Live path | Analysis | Stream | Verdict then |
|---|---|---|---|---|
| A | PostgreSQL | ClickHouse | Kafka to both | Recommended |
| B | TimescaleDB (one database, native time partitioning and continuous aggregates) | same | optional | Simpler |
| C | ClickHouse only | ClickHouse | | Not recommended (poor OLTP) |

Stream processing: Kafka to Flink for at-least-once aggregation into trap_patterns, or a custom consumer.

## What the prototype did instead, and why

In-process Python; a hash-chained, append-only audit log on disk; an HTTP demo server for limit reads and operator decisions; trap events read from the Rust kernel's output. No database, no queue.

The reason is the one the critique gave: the interesting claim is the governance model (asymmetric authority, an interpretation layer, validation), not the storage. Every experiment in five trace rounds ran against this in-process version at 200,000 tasks per trace in seconds. The database would have added nothing to the answers and plenty to the surface area.

## What the evidence now says the storage must provide

1. **A one-row live read by key.** The kernel reads the agent's current limit before an execution (`GET /boundaries/stack.agent.<id>.deadline_ns`) and caches it for a slice. This is the only hot-path read. It does not need the time-bucketed trap counts; it needs the limit. Any key-value store or a single indexed table does it.
2. **An append-only, hash-chained decision log with write-ahead semantics.** The limit does not change until its log entry is durable. Startup replays and verifies the log. This is the source of truth, not trap_events.
3. **An anchor outside the writer's reach.** The log's head hash must be stored where the governor's process can append but not rewrite: a file owned by another user with the append-only attribute, an append-only bucket, or a second service. Without it a full rewrite of the log verifies. The database that holds the log must not be the only place its head is kept.
4. **Per-slice counters for validation**: tasks, deadline traps, canary tasks, canary traps, per agent per slice. Small, and they age out.
5. **Idempotency keys on every outside signal**, surviving a restart, so a retried tightening never applies twice.
6. **Cold analysis of raw trap events**, which can be a columnar store and can lag by minutes.

## Recommendation

- Start with option B or plain PostgreSQL: the audit log, the boundary table, the per-slice counters and the pending-review queue in one database, with a materialisation job for the counters (`materialization-correctness.md`).
- Keep the anchor out of that database. A second, append-only location is not optional.
- Add Kafka only when enforcement runs in more than one region and the log has to be replicated; add ClickHouse only when cold analysis of raw traps outgrows PostgreSQL.
- Put TLS in front of every HTTP path before a second machine is involved; the prototype has none.
- Do not build a learning store for `effectiveness_score` as a float. Store the last verdict with its bounded cost and benefit numbers.
