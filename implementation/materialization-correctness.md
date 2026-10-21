# Materialization Correctness: The Checkpoint Bug and the Prototype's Equivalents

**Status:** The white paper's pseudocode has a known unsafe step. The fix is standard. The prototype avoids the problem by construction, in ways worth copying.

---

## The bug

```
since = SELECT MAX(digest_timestamp) FROM trap_patterns
events = SELECT * FROM trap_events WHERE timestamp > since
```

`digest_timestamp` is per pattern row. If pattern A was digested at 12:05 and pattern B at 12:00, `MAX` returns 12:05 and the events for B between 12:00 and 12:05 are never aggregated. Any run that updates some rows and not others (a crash, a partial batch, a slow partition) leaves a permanent hole that the next run skips past.

## The fix

A separate watermark that belongs to the job, not to the rows:

```
materialization_runs
  run_id, started_at, completed_at, source_watermark, status, checksum
```

- Read events with `timestamp > last completed run's source_watermark`.
- Aggregate and upsert into trap_patterns inside one transaction.
- Advance the watermark in the same transaction, only on success.
- Upserts must be idempotent (keyed on context hash, boundary, bucket) so that a reprocessed batch after a crash produces the same rows, not doubled counts.
- Use the event id, which is monotonic, as the watermark rather than a timestamp, to avoid clock skew between writers.

With at-least-once stream processing (Flink from a checkpoint, or a consumer replaying an offset) the same two properties, idempotent upsert and a single transactional watermark, are what make a replay harmless.

## What the prototype does, which amounts to the same thing

| Concern | Prototype mechanism | Where |
|---|---|---|
| A change applied twice after a crash | Every outside signal carries an idempotency key; the key is written into the boundary-version entry in the audit log, so a retried call after a restart returns the remembered result instead of tightening again | `src/governance/signals.py`, `tests/test_signal_interface.py` |
| A change recorded but not applied, or applied but not recorded | Write-ahead: the store writes the boundary version to the audit log before the limit takes effect; if the write fails the change does not happen | `docs/DURABLE_STATE.md` |
| State drifting from the log | At startup the log is verified and replayed to rebuild every limit, the breaker count and the decision list; there is no separate state to get out of sync | `Governor._restore` |
| A duplicate trap event | Trap ids are deduplicated per agent and reason window | `integrations/stack_traps.py` |
| Evidence counted twice toward a tightening | The trap window is cleared after each tightening and after a drift hold lifts through recovery, so a tightening always needs a fresh pattern | round 4 of `docs/TRACE_TEST_PLAN.md` found and fixed the recovery case |
| A verdict issued on a slice whose traps are not in yet | The validation review runs over completed slices only, before the new slice is opened | round 5 |

## Two further correctness notes for a real build

1. **Rate windows must be task-based, not time-based.** The rate rule counts traps among the last N tasks the agent ran, which the kernel reports. A time-based window would let an idle agent's old traps look like a high rate. If the kernel stops reporting activity, nothing tightens (fails safe).
2. **The review cadence is measured from the last tightening.** A boundary that tightens three times in three slices gets its first verdict four slices after the third, not the first. That is why slow-from-start agents in the traces were confirmed at slice 6 or 7, not 4. If a build wants faster confirmation it should measure from the first tightening and accept noisier early verdicts, or run the canary from the first tightening regardless.
