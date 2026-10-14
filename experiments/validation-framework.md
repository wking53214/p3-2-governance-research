# Validation Framework: Proving a Tightening Helped, As Far As It Can Be Proved

**Status:** Built for deadline tightenings, pre-registered as criterion 10, run twice on fresh seeds. It fails its own speed bar. Evidence: `experimental/docs/TRACE_TEST_PLAN.md` (round 5), `experimental/docs/SUCCESS_CRITERIA.md` (criterion 10), `experimental/integrations/stack_traps.py` (`_review_if_due`).

---

## Question C, restated

A tightening reduces violations by construction: a lower limit traps more. That tells you nothing. The question is whether the tasks it newly traps were ones you wanted trapped (hung, runaway) or ones that would have completed (legitimate work, now killed and redone). No oracle exists for that at the time of the decision.

## The signal: a canary

While an agent's deadline is tightened, every tenth of its tasks keeps the original deadline. That is the A/B test the research questions asked for. Two things are then observable with no labels and no baseline:

- the trap share of the main tasks at the tightened deadline D1,
- the trap share of the canary tasks at the original deadline D0.

The difference is the collateral: tasks the tightening kills that would have completed under D0. The canary share is an upper bound on the hung share (every trap at D0 might be a hang; none of them might be).

## The verdict, in one currency

- cost per task = collateral x D1 (a newly killed task burned the whole tightened deadline)
- benefit per task, at most = canary share x (D0 - D1) (each hang saved earns only the cut)
- **DEGRADED** if the lower confidence bound of the cost exceeds the upper bound of the benefit: a person gets a restore request, further automatic tightening of that boundary stops, nothing is loosened.
- **IMPROVED** if the upper bound of the cost is under the lower bound of the benefit.
- **UNCHANGED** when the bounds do not separate. This means "cannot tell", not "no effect".
- Bounds are normal-approximation at two standard errors, rule of three for a zero count, over every slice since the tightening.

Verdicts go to the audit log and the governor's validation store. `effectiveness_score` in the schema is this verdict with its two bounded numbers.

## What happened

| | First run (no bounds, 4-slice window) | Second run (bounds, cumulative window) |
|---|---|---|
| Correct runaway tightenings wrongly declared DEGRADED | 70 of 80 | 7 of 80 |
| Restore requests on tightened agent-seeds | 355 of 400 | 182 of 408 |
| Slow-from-start agents (gross harm) flagged | 60 of 60 | 60 of 60, 48 within 6 slices |
| Wide-tail agent at a 1.5% natural timeout rate flagged | 20 of 20, 3 within 6 slices | 20 of 20, 1 within 6 slices |
| Late-released legitimate regressions flagged | most | 3 of 28 (not enough slices left) |
| Correct tightenings confirmed IMPROVED | often, by noise | 13 of 80 |
| Criterion 10 | FAIL | FAIL, on confirmation speed only |

## What the arithmetic says

One newly killed task costs the whole tightened deadline; one hang saved earns only the cut. At the first 10% step, IMPROVED needs the collateral shown to be under about a ninth of the hang share, which takes tens of thousands of canary tasks. DEGRADED needs a collateral of several points to resolve within a few slices. So at a 10% canary the layer confirms gross harm fast, marginal harm late, and benefit almost never. A 30% canary resolves everything about three times faster and gives up about a third of the saving. That trade is the operator's to make; it was not tuned here.

## Limits, deliberately

- **Blind to intent.** An attacker's inflated-but-completing tasks are collateral like anyone's. About half the inflating attackers get a restore request a person must refuse.
- **Cumulative windows cannot see an episode end.** A tightening that stopped buying anything after a runaway episode ends is not noticed (1 of 20).
- **Deadline only.** Token and memory caps showed no benefit to measure in the traces.
- **The canary costs the saving it measures.** 10% of hangs run to the original deadline.
- **Not real traffic.**

## What passing would look like

Criterion 10(d) asks for 80% of wrong tightenings confirmed within 6 slices. The bounded layer reaches 42%. Either a larger canary (priced above), or a looser bar written down as a judgment call, or a richer signal (does the killed task complete on retry) that resolves faster than counts. None of these is a tuning problem on synthetic seeds.
