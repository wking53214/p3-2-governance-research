# Program: priced, accountable adaptive limiting, and a benchmark to score it on

**Status:** pre-registered research program, written 2026-10-15 before any of its experiments run. It records what thirteen rounds and a red team established, what the thesis becomes in their light, which existing capabilities in this portfolio are reused, the open research problem at its center, and the predictions each phase must meet. Nothing here is a result.

**Provenance of the capability survey.** Forty public repositories and the private white-paper drafts were surveyed on 2026-10-15 by three independent read-only reviewers plus a keyword sweep. Their file-level findings are reported here as they stated them and were not all re-verified line by line; each phase below re-verifies what it reuses before building on it. A capability counts only when it is working, tested code; README claims without code behind them are listed as vocabulary.

---

## 1. Where the evidence stands

Thirteen pre-registered replay rounds, six hostile reviews and two fixes to the invariant's enforcement (`experimental/docs/RED_TEAM.md`, `experimental/docs/TRACE_TEST_PLAN.md`) settled the following.

| Claim | Standing |
|---|---|
| Automation can only tighten a limit; loosening needs a named human's single-use grant | Holds under fuzzing, soak, thirteen rounds and a red team. Two real enforcement holes found and fixed (exact comparison of limits; operator actions refused without a registry). |
| Capping automation (three steps, then stop) is what keeps honest programs safe | Holds. Pooled honest cost 17.8 points uncapped, 2.0 capped; replicated on fresh seeds, a held-out roster and two duration shapes. |
| Interpreting the signal (drift hold, excess release, truncation-aware slowdown fit, expected-load hook) is the other half | Holds. Each rule isolated and measured; round 13 showed they travel intact into a two-way limiter. |
| The one-way direction itself protects | **Withdrawn.** Its measured benefit was a cold-start artefact of the rate window. A two-way limiter with the same rules and hysteresis on its increase reproduces the ratchet on every cell of five rosters, with no human decisions. |
| Honest programs are never tightened | **Conditional on headroom.** True with the deadline 4 to 17 times above honest medians; at three times, every honest program with a tail is tightened. |
| The loop saves time | **False as measured.** Priced as hung time saved minus honest work destroyed, the loop loses about 3% of all task time at the original deadline and 6% at a tight one. Only sustained runaways are a net win. |
| The human gate's price | 216 decisions per 500 program-runs on the reference roster, 62 of them on programs that should not have been tightened. |

Everything above was measured on synthetic traces with no time structure. Nothing has run on real traffic.

## 2. The thesis, re-aimed

The original thesis put the safety in the direction of change. The data put it in the bound and the interpretation. The re-aimed thesis:

> An adaptive limiter for autonomous programs should (a) price every tightening before making it, in the currency of time destroyed against time saved, (b) treat every automatic change as an obligation that must be settled by later evidence or abandoned for a stated reason, (c) report which classes of program it harms, and (d) be scored on a public benchmark whose hard cases are the ones that broke the first design. The one-way rule remains as an audit and authority property, not as a protection.

What is new is not any mechanism. It is the combination, the accountability loop, and the measured decomposition behind it. The industry has rate limiters, circuit breakers and approval gates; it does not have a limiter that prices its own actions, owes an outcome on each one, and is benchmarked on net time rather than on counts of things blocked.

## 3. Capabilities reused from this portfolio

| Role in the program | Source | State as reported by the survey | Adapter | Caveat |
|---|---|---|---|---|
| **Outcome obligations**: every tightening opens an obligation with a horizon; it resolves favorable or unfavorable, stays open for a named reason, or is abandoned; overdue is computed, never stored | `interconnected_delta` (`delta/obligation.py`, `decision_obligations.py`); `sentinel_os` (`outcome_v1.py`, twin `obligation_ledger`, shadow-run record and score, harm events) | Working, tested; delta is in-memory, sentinel_os persists to Postgres | Wrap a tightening as a decision with a stable fingerprint; rule such as `tightening_effect@1h`; resolve by counterfactual replay with provenance `verified`, else `estimated` with the method named; add persistence by appending to the limiter's own hash-chained audit log | Yes/no verdict only, no effect size; outcomes are caller-trusted, so the replay that resolves them must itself be audited |
| **Disparate-impact screen**: which program classes bear the harm | `interconnected_delta` (`delta/fairness.py`, four-fifths rule); `Reporting` (`compas_audit.py`: ratio against a reference group with intervals and a minimum group size, matched-rate comparison) | Working, tested; used on the COMPAS data | Group = program class (team or profile); favorable = not harmed by tightening; report at matched tightening rates | A screening heuristic, not a significance test; needs the group floor from Reporting's version |
| **Benchmark harness** | `SWIZZLE` lab (`TargetAdapter`, `GroundTruth` frozen before the run, oracles that cannot import the adapter, `Fitness` ranked by severity, `metrics.scorecard` with every rate carrying its denominator, `holdout` frozen by digest with contamination checks, `standing` where a resolved claim can be overturned by a probe, `differential` baseline-against-candidate) | Working, tested; every current oracle and specimen is about code repositories | Replace the repository genome with a trace genome (the generator in `experimental/experiments/gen_trace.py` and its hard cases); map scan/act to "run the limiter over the trace and return its decisions"; write limiter oracles: missed runaway is unsafe, needless tightening is over-caution; add net time as a metric | Nothing in SWIZZLE measures time; that is added, not adapted |
| **Answer-key integrity** | `ASSAY` (`assay_production/verification.py`: hash on every specimen, four copies of the key that must agree, verification level derived rather than typed, machine-readable summary line, sandbox) | Working, tested; specimens are damaged code, not traces | Reuse the mechanism on trace specimens: each committed trace carries its hash, its ground-truth labels, and whether its labels are verified by replay or asserted by a person | Only the mechanism carries over |
| **Recovery rules for the two-way bridge** | `URE` (`dwell.py`: a baseline that shrinks fast, grows slowly, grows slower still under suspicion, with a hard ceiling and a walk-up test; `recovery.py`: escalate at once, relax only after N consecutive proposals agree) | Working, tested, stdlib | Convert both from observation counts to elapsed task time; use as the increase rule in `BridgeTwoWay`; keep the walk-up test as a regression test | The "Lyapunov" scoring around them is vocabulary and is not reused |
| **Red-team protocol** | `SWIZZLE` governor scenarios (held / violated / not run; a crash is never "held"; companion scenarios isolate one guard; two-sided, with over-caution scenarios) | Working | Scenarios over limiter decisions | Scenarios are hand-written |
| **Pre-registered claims** | git (predictions commit, then results commit, as rounds 12 and 13 did); `CCC` for the claim record with human-only promotion and final rejection | CCC is working but has no tamper evidence or trusted time | Keep git as the anchor; CCC holds the claim text, its basis, and its SUPPORTS or CONTRADICTS link to the result | CCC alone cannot prove a prediction preceded a run |

**Patterns copied, not code.** JUDGE's ordering (harm first, then missing evidence, then accept; unknown is never approved) as the benchmark's acceptance gate. HERALD's asymmetric calibration (dangerous failures block, over-caution is tracked). WARDEN's held-result digest as the human gate on an unverified automatic change. observe-perceive's rule that a threshold moves in the safe direction by default and the other way only by explicit permission.

**Vocabulary only, not reused.** The "Lyapunov" and "energy" controllers in `fortress-kernel` and `URE` (weighted scores with thresholds; one relaxes faster than it tightens). `Conservation_Kernel` (categorical provenance checks, not an accounting of loss). `GSA`'s load figures (a mock generator, in-process, never deployed). Survival and hazard hits in the archived conversations.

**Not available anywhere in the portfolio.** Real traffic. Survival or hazard estimation. Handling of censored durations. These are built new or obtained outside.

## 4. The open research problem: tightening censors its evidence

A limiter learns a program's duration distribution from the tasks that complete. Once it tightens the deadline, the tasks it kills never complete, so they leave the sample. The learned distribution drifts shorter, which argues for tightening further. The same blindness defeats any false-kill feedback: the label of a killed task (hung or honest) is never observed.

Every priced-tightening rule must therefore estimate, not observe, what a killed task would have done. The program treats this as its central question, and it is a research question, not a plumbing one. Candidate approaches, to be chosen by a pre-registered comparison in phase 2: extrapolating the completed distribution past the deadline with a parametric tail; treating kills as right-censored observations and estimating with a survival method; and the canary already built (a share of tasks run at the original deadline, which observes the tail directly at a known cost). The canary is the only one that observes rather than estimates, and its cost is already measured (about a third of the saving at a 30% share).

## 5. Phases, with what each must show

Each phase is pre-registered in its own commit before any run, as rounds 12 and 13 were. Each ends with a results commit and, at the program's end, another independent red team.

### Phase 1: the benchmark (no new mechanism)

Build the harness from SWIZZLE's and ASSAY's mechanisms over the existing trace generator. Specimens: the hard cases that broke the first design (cold-start pulsers, patient pulsers, inflate-first attackers, headroom at 100, 75 and 60 ms, bimodal and Pareto durations, the tight honest agent, slow-from-start agents), each with hashed ground truth and a verified-or-asserted label. Metrics: net time with burn and with rerun, honest cost in points with and without the slow-from-start pair, per-seed medians beside pooled numbers, disparate impact by program class, operator decisions, and the red-team scenario table. Systems under test on day one: the ratchet, the capped two-way with the drift hold, the two-way bridge with hysteresis, and a plain static deadline as the floor.

Predictions, committed with the harness before it is run on the four systems:
- P1.1: the scorecard reproduces every number in rounds 12 and 13 for the three limiters to the printed precision.
- P1.2: the static deadline is net-positive on no roster; every limiter is net-negative on every roster except on sustained runaways, as round 12 found.
- P1.3: the disparate-impact screen flags the slow-from-start class and the tail class under every limiter at 100 ms, and every honest class at 60 ms.

### Phase 2: priced tightening

Add to the two-way bridge a rule that tightens only when the predicted time saved exceeds the predicted honest work destroyed, under each of the three censoring treatments in section 4, with URE's recovery rules as the increase side.

Predictions:
- P2.1: on the reference roster at 100 ms, a priced limiter declines to tighten the period-8 and period-16 pulsers and the low-and-slow attacker (net losses in round 12) and still tightens the sustained runaways, so its roster net time is positive where every round-12 limiter was negative.
- P2.2: at 60 ms, the priced limiter's honest cost is at most half the unpriced bridge's (8.4 points) and its net time is at worst zero.
- P2.3: the parametric-tail treatment and the survival treatment disagree with the canary by more at 60 ms than at 100 ms, and the direction of their error is toward over-tightening (the censoring bias of section 4).
- P2.4: the price of the canary treatment, in saving given up, is within 5 points of round 6's measurement.

Falsifiers: P2.1 failing means pricing does not separate the agents the net-time metric says it should; P2.3 failing in the other direction means the censoring argument is wrong.

### Phase 3: accountable tightening

Every tightening opens an obligation (delta's model, persisted in the limiter's audit log). It is resolved by counterfactual replay of the same slice at the original deadline, favorable when the realized net time is positive, unfavorable when negative, open for a named reason when the replay is not yet possible, abandoned when the program disappears. Unfavorable obligations feed a bounded trim on the price threshold, as GSA's false-discovery loop does but durably and for allowed as well as blocked actions.

Predictions:
- P3.1: on the reference roster, every tightening's obligation is settled within two slices; the unfavorable share under the priced limiter is under 10%, under the unpriced bridge over 40%.
- P3.2: the trim moves the price threshold monotonically in the direction that reduces the unfavorable share on fresh seeds, and never below the floor that keeps sustained runaways tightened.
- P3.3: the human is asked for nothing on honest programs; the only decisions presented are abandonments and overdue obligations, and their count is under one per hundred thousand tasks.

### Phase 4: real traffic

The only path in the portfolio is `GSA` with a real generator behind it, per-request token and memory capture, agent id and expected-load flag on the request, a JSONL exporter over its trace table, and durable feedback. Until real agent traffic flows through it, this phase has no predictions, only a gate: no claim from phases 1 to 3 is stated as more than a synthetic result until it is replicated here.

### Phase 5: red team, again

The same protocol as `experimental/docs/RED_TEAM.md`, with fresh reviewers given the code and data and not the conclusions, with a standing brief to attack the censoring treatment and the benchmark's ground truth first.

## 6. What this program does not claim

- That any single mechanism is new. Rate limiting, breakers, approval gates, survival estimation, four-fifths screening and outcome ledgers all exist.
- That the one-way rule protects anything measurable. It is kept for attributability and because no workload can move a limit up through it.
- That synthetic results predict production. Phase 4 is the gate.
- That pricing solves censoring. Phase 2 measures how badly each treatment is fooled; the honest outcome may be that only the canary is trustworthy, at its measured cost.

## 7. Dependencies and risks

- Phase 1 depends on nothing outside this portfolio and can start now.
- Phase 2 depends on new estimation code; the censoring problem may have no estimator good enough without the canary.
- Phase 3 depends on phases 1 and 2.
- Phase 4 depends on access to real agent workloads, which this portfolio does not contain. This is the program's binding constraint and has been since the first day.
- The reviewers' reports on the reused repositories were read-only and are not a substitute for running their test suites before reuse; phase 1 begins by doing so.
