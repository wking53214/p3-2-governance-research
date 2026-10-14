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

## What the direction itself buys, measured (experimental rounds 9 and 10)

Head to head with a two-way adaptive limiter (AIMD) on the same traces and seeds, taken apart feature by feature:

| Feature | What it buys | Evidence |
|---|---|---|
| The cap (three automatic steps, then stop) | Honest agents are not ground to the floor: pooled legitimate cost 17.8 points without it, 4.0 with it. It also caps the saving at about 26% on anything sustained, where a limiter with no cap saves 65% | round 9b |
| The drift hold | Honest regressions and drifts: 8.4 points under cap alone, 0.0 with the hold | round 10a, 10c |
| The excess release | Attackers who inflate their completions first: a drift hold alone gives up the whole saving on them (0% against 19 to 21%) | round 10c |
| The one-way rule | An attacker who can wait out the limiter's recovery keeps its budget under a two-way limiter and loses it under the ratchet. It matters exactly when `(period - 1) x increase >= 0.10`, which predicted 10 of 10 cells of a sweep | round 9, 10b |
| The human gate | Not measurable in a replay; it is what turns the cap's "stop" into "stop until someone looks", and it is the price of the direction | |

A two-way limiter with the cap and the drift hold matches the ratchet on every honest agent to within a point. So the direction is not the source of safety; bounding automation and interpreting the signal are. The direction's contribution is narrow, real, and stated as a formula.

## The honest statement

The decomposition above is the contribution; the ratchet is one point in that space. It relocates human effort from watching every execution to adjudicating a short list of held tightenings and restore requests. It does not reduce the need for judgment; it moves it, and it keeps the system safe while the judgment is pending.
