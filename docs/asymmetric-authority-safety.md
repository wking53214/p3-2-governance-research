# Asymmetric Authority: Why Tighten-Only Matters, and What It Costs

**Status:** The one part of the proposal that has been built, tested and not falsified. Evidence lives in the `experimental` repository (paths below; PR #5 until merged).

---

## The invariant

```
Observed enforcement pressure  ->  automatic tightening (within a budget)
Absence of pressure            ->  no automatic loosening, ever
Loosening                      ->  a named human, one single-use grant per change
Disabling                      ->  not implemented (there is no disable path to guard)
```

The point is not that tightening is always right. It is that the easiest way to game an adaptive system, teaching it that constraints are unnecessary by behaving well, has no effect here. Nothing an untrusted workload does can move a limit up.

## How it is enforced, not just stated

| Mechanism | What it stops | Where |
|---|---|---|
| Loosening needs a single-use grant the authority issues only on a named operator's approval, checked against the boundary's actual limit | Mislabelled proposals (T6), side-door loosening through rollback, direct store writes or replayed grants (T7) | `src/governance/grant.py`, `boundary.py`, `tests/test_invariant_attacks.py` |
| 20% usability floor | Automatic tightening to zero | `governor.py` |
| Circuit breaker: 3 automatic tightenings per boundary, then held until a person acknowledges | Tightening used as denial of service (T3); a buggy signal source spending more than about 27% of a limit (T14) | `docs/THREAT_MODEL.md` |
| Hash-chained audit log with write-ahead boundary versions; limits rebuilt from it at restart; a tampered log refuses startup | A restart resetting the ratchet (T13); silent edits to history (T8) | `docs/DURABLE_STATE.md`, `src/governance/audit_log.py` |
| Anchor store: the log's head published after every append, fail closed | A full rewrite or truncation of the log (T8), if the anchor lives out of the process's reach | `src/governance/anchor.py` |
| Outside signals (`propose_tightening`, `request_loosening`) can only tighten or ask | A monitor or kernel driving a limit up | `docs/INTEGRATION.md`, `tests/test_signal_interface.py` |

Tested by: a fuzzing attacker that only observes (never raised a limit), 36 of 36 soak runs with the integrity check green, and the trap-loop replays, where no run ever produced a loosening without a grant. Pre-registered as criterion 1 and re-checked by criteria 7, 8 and 10(c) in `docs/SUCCESS_CRITERIA.md`.

## What it does not cover

- Code running inside the governor's process (A4): edits are evident, not prevented.
- A network attacker (A5): no TLS on the demo paths.
- A malicious or mistaken operator (A6): decisions are logged, not constrained.
- Operator identity is a shared secret, not a signature (T9).

## What it costs

1. **Availability.** Tightening is the safe direction for security and the unsafe one for availability. The breaker caps the damage at about 27% of a limit without a human; a boundary with less headroom than that is still hurt.
2. **Constraint accumulation.** Limits only go down on their own, so someone has to review and restore them. The trap-loop validation layer files restore requests when a tightening is confirmed harmful; everything else waits for a periodic review. This is a feature, and it is not "no human oversight".
3. **The benefit is bounded by the same cap.** On a runaway task the loop saves at most the breaker's arithmetic, 27% of the time those tasks burn (23% with the validation canary). A person with judgment does better on any sustained attack. The loop wins only in the window before a person responds.

## What the direction itself buys, measured (experimental rounds 9 to 12 and the red team)

Head to head with a two-way adaptive limiter on the same traces and seeds, taken apart feature by feature:

| Feature | What it buys | Evidence |
|---|---|---|
| The cap (three automatic steps, then stop) | Honest agents are not ground to the floor: pooled legitimate cost 17.8 points without it, 2.0 with it (97% of that 2.0 is two agents slow from their first task, which any capped policy cuts identically). It also caps the saving at about 26% on anything sustained, where a limiter with no cap saves 65% | rounds 9, 9b, 12c |
| The drift hold | Honest regressions and drifts: 8.4 points under cap alone, 0.0 with the hold at the 100 ms deadline | rounds 10a, 10c |
| The truncation-aware slowdown fit | At 75 ms of headroom the ratchet's bridge holds honest regressions at 1.5 points where a hold on the raw median costs 10 | round 12b |
| The excess release | Attackers who inflate their completions first: a drift hold alone gives up the whole saving on them (0% against 19 to 21%) | rounds 10c, 12c |
| The one-way rule | **Nothing measurable on these traces.** Rounds 9 to 11 showed a benefit against patient pulsers; the red team showed those pulsers were caught in the limiter's first 250 tasks, where its rate window had not filled. A pulser that waits is not caught by the ratchet (2.5% and 0% saved), and a two-way limiter with 16 slices of hysteresis on its increase catches it at every phase (18 to 19%, 11%) at the same honest cost. With the interpretation layer held fixed (the ratchet's own bridge run two-way), the direction is worth 3 points of saving on one patient pulser under 8 slices of hysteresis and nothing under 16, with every other cell of five rosters identical | rounds 12a, 13 |
| The human gate | 216 decisions per 500 agent-seeds, 62 of them on agents that should not have been tightened. A capped two-way limiter asks for none and reaches the same honest-agent cost | round 11d, 12c |

So the direction is not the source of safety; bounding automation and interpreting the signal are, and a two-way limiter carrying the same interpretation rules reproduces the ratchet exactly with no human in the loop (round 13). The "direction matters" claim made after round 10 is withdrawn. What remains to the one-way rule is unmeasured by replay: every change is attributable, and nothing an untrusted workload does can move a limit up.

Limits found in rounds 11 and 12 and by the red team (`experimental/docs/RED_TEAM.md`): under a heavy-tailed duration distribution the drift hold weakens; the honest-agent results assume the deadline sits 4 to 17 times above honest medians, and at three times the slowest honest median every policy tightens every honest agent with a tail; priced in time (hung time saved minus the legitimate work the tightened deadline destroys) the loop is a net loss of about 3% of all task time at 100 ms and 6% at 60 ms, and only sustained runaways are a net win. The breaker cap is a dial: each automatic step buys about 9 points of saving on sustained attackers for about 0.7 points of pooled honest cost.

## The honest statement

The decomposition above is the contribution; the ratchet is one point in that space, and on these traces its direction is not the part that protects anyone. It relocates human effort from watching every execution to adjudicating a short list of held tightenings and restore requests. It does not reduce the need for judgment; it moves it, and it keeps the system safe while the judgment is pending.
