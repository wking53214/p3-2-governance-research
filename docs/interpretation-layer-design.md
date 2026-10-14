# The Interpretation Layer: What Was Built, What It Separates, What It Cannot

**Status:** Built and tested on synthetic traces (five rounds, 20 seeds each in the last two). Not tested on real traffic. Evidence: `experimental/docs/TRACE_TEST_PLAN.md`, `experimental/integrations/stack_traps.py`.

---

## The problem it has to solve

The critique's list: a violation at a 100 ms deadline can mean the boundary is too tight, the workload needs optimising, the workload is malicious, the environment is degraded, the metric is wrong, or another constraint is causing it. "Violation, therefore tighten" is wrong as a rule. Something has to sit between the trap event and the decision.

## What sits there now

Each rule answers one of those interpretations with the cheapest signal that can answer it.

| Rule | Question it asks | Signal | What it separates | Where it fails |
|---|---|---|---|---|
| Expected-load hook | Was this heavy task declared in advance? | A scheduler flag on the task | Scheduled batch jobs (never tightened, 0 of 60 agent-seeds) | Anyone who can register a pattern can excuse their own attack unless registration is an audited operator action (T4) |
| Repeated pattern (default) | Is this a one-off? | 5 traps in the last 50 | Single events | Tightens anything with a tail, however small: a healthy agent at 0.05% timeouts reached the cap in the first closed loop |
| Rate rule | Is the trap share material? | Traps among the agent's last 2,000 tasks, at least 1% | Healthy agents with tails at or below 0.14% (0 of 180 agent-seeds tightened) from runaways at 2% and up (100%) | Any agent whose natural tail sits above the threshold: a 1.5% tail is tightened every time. The 1% is a workload parameter; the sweep found labelled agents at 0.6%, 0.8%, 1.5% and 2% on both sides of it |
| Drift rule | Did the agent's completed tasks slow down too? | Median completed duration against the agent's own first five slices | A regression (completions slowed, hold and ask a person) from a runaway (completions unchanged, tighten): 0 of 11 legitimate regressions tightened under the plain hold | Needs history, so an agent slow from its first task is tightened anyway (accepted limit). Opens T15: an attacker that slows its completions first is held for the whole attack |
| Excess release | Does the slowdown explain the trap share? | Five completed-duration quantiles, fitted for the slowdown under deadline truncation, against the observed trap share | Closes most of T15 (120 of 120 inflate-first attackers tightened, 3 to 7 slices late) | Falsely releases about 1 legitimate regression or drift in 8, late and cheaply. Estimator saturates near 4x. Assumes the old distribution only scales |
| Recovery clearing | Did the hold just lift because completions recovered? | The drift state flipping back | Stops the slow period's traps tightening a regression right after it recovers (found in round 4: 11 of 20 seeds, now 0) | None found yet |

Order of evaluation: expected-load, pattern, rate, drift and excess, then the validation layer afterwards (`experiments/validation-framework.md`).

## What it cannot do, from trap counts

- Tell a legitimately wide-tailed agent from a mild runaway. Both are "2% of tasks exceed the deadline". No count rule separates them; the validation layer catches the wide-tail case afterwards, late.
- Tell an attacker's inflated-but-completing tasks from a legitimate regression. The drift rule holds both; the excess rule releases both when hangs begin.
- Judge an agent with no history.

## What would do better, and what it needs

A signal that is not a count: whether a killed task completes on retry, what the task was, which class of agent it belongs to. Per-class thresholds set by an operator are the cheap version and need no new data. The richer signals need kernel support and real traffic to design against.

## Recommended configuration today

```
TrapPolicy(min_rate=0.01, drift_factor=1.5, drift_recent_reports=1, drift_excess=True,
           reasons=("deadline_exceeded",), validate_after=4)
```

Deadline only: tightening token and memory caps added collateral and no benefit in the traces (a leak exceeds any cap). The drift factor is flat between 1.3 and 1.5 and breaks at 2.0. The rate threshold is on an edge in both directions and has to be set for the workload.
