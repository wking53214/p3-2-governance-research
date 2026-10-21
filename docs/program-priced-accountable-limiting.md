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

---

## 8. Novelty check against the re-aimed thesis (2026-10-15)

Twelve web searches in extended mode, aimed at the four pillars of section 2 rather than at the old thesis. US-only index; preprints dominate; several hits are vendor or practitioner posts and are marked so. Absence of a hit is weak evidence. This section exists so that a referee meets these works here first.

### Pillar (a): pricing a tightening in time, under censoring

| Adjacent work | What it has | What it lacks relative to this program |
|---|---|---|
| Straggler mitigation by delayed relaunch ([arXiv 1710.00414](https://arxiv.org/pdf/1710.00414)); optimal redundancy in master-worker clusters ([arXiv 1906.05345](https://arxiv.org/pdf/1906.05345)); safe duplication for tail latency ([arXiv 1905.13352](https://arxiv.org/pdf/1905.13352)) | Explicit cost models for killing or duplicating a slow task: relaunch time as a function of tail heaviness; marginal latency benefit set against marginal resource cost; the observation that aggressive schemes destabilize at load | Cooperative stragglers, not adversarial programs; cost is latency or redundancy, not honest work destroyed; no learning from censored completions; no accountability after the fact |
| Deadline-aware admission control: Cucumber ([arXiv 2205.02895](https://arxiv.org/pdf/2205.02895)), LegoDiffusion ([arXiv 2604.08123](https://arxiv.org/pdf/2604.08123)); a practitioner piece on LLM serving as deadline-aware admission control ([wtfistrending](https://wtfistrending.com/trends/llm-serving-deadline-aware-admission-control/), not a study) | Reject up front when a deadline would be missed; preempt or migrate only above a measured threshold | Decides before work starts; nothing about tightening a running program's limit or pricing the kill of work already done |
| Timeouts as censoring: a production guide ([OneUptime](https://oneuptime.com/blog/post/2026-07-28-choose-production-http-timeouts-latency-percentiles/view), practitioner); censoring-aware survival losses ([KAN-AFT, arXiv 2512.20305](https://arxiv.org/pdf/2512.20305)); survival with censored reporting delays ([arXiv 2510.04421](https://arxiv.org/html/2510.04421)); adaptive Raft timeouts as online optimization ([BALLAST, arXiv 2512.21165](https://arxiv.org/pdf/2512.21165)) | The premise of section 4 is recognized in practice (a 500 ms timeout hides whether p99.9 is 520 ms or 20 s; estimate the tail with a canary); general survival methods handle right-censoring; one adaptive-timeout paper treats the choice as online optimization | No published estimator of a task's hidden tail inside an adaptive limiter; no statement that the limiter's own action biases its baseline; a Kaplan-Meier approach to latency appears only in a closed GitHub issue and a patent analogy |

**Verdict for (a):** the cost-model vocabulary exists in cluster scheduling, and the censoring premise is acknowledged in practice. No found work combines them inside an adaptive limiter for autonomous programs, and none states or measures the self-censoring loop of section 4. Phase 2 cites the relaunch and duplication papers for its cost model and the survival literature for its estimators.

### Pillar (b): every automatic change owes an outcome

| Adjacent work | What it has | What it lacks |
|---|---|---|
| Post-Rejection Follow-up Sampling ([arXiv 2606.08228](https://arxiv.org/html/2606.08228); dataset and code on Zenodo and GitHub; single independent author, preprint) and its companions RED-2400 ([arXiv 2605.12151](https://arxiv.org/pdf/2605.12151)) and outcome-classified precision auditing ([arXiv 2607.02830](https://arxiv.org/pdf/2607.02830)) | **The closest match found.** A separate tracker follows every rejected trade candidate on the same live feed to a fixed horizon, so filter precision is judged against what actually happened to what was rejected; a public benchmark of rejection events with outcome labels | Trading filters, not resource limits; observational, no control loop that feeds the outcomes back; no notion of an obligation that can stay open for a named reason or be abandoned |
| Agent Control Protocol ([arXiv 2603.18829](https://arxiv.org/pdf/2603.18829)) | Every approve, escalate or deny decision written to a chained ledger before the action runs, serving both enforcement and post-hoc verification | Records the decision, not its later outcome; no obligation, no resolution, no feedback to the policy |
| Protocol-Driven Development ([arXiv 2605.12981](https://arxiv.org/html/2605.12981v3)) | Append-only dynamic evidence ledgers holding runtime observations, violations and remediation outcomes; a runtime layer that blocks, quarantines, rolls back or degrades | Software-invariant governance, not resource limiting; outcomes are of remediations, not of the governor's own automatic decisions |
| Decision-ledger vendor material ([Raktim Singh](https://www.raktimsingh.com/decision-ledger-enterprise-ai/), [Acceldata](https://www.acceldata.io/blog/the-hidden-risks-of-over-automating-data-governance), [Fast Company](https://www.fastcompany.com/91466077/the-hidden-advantage-of-the-runtime-control-plane); a delegated-authority patent application) | "Decision receipts" with inputs, policy, timestamp, outcome; accountability records of which constraints applied | Prose and product claims; no measurement, no open-resolve-abandon lifecycle |
| Self-adaptive systems (MAPE-K): formal verification of loops ([Arcaini et al.](https://cs.unibg.it/scandurra/papers/seams2015_cameraReady.pdf)), decentralized control ([ACM TAAS](https://dl.acm.org/doi/10.1145/3019598)), RDMSim exemplar ([arXiv 2105.01978](https://arxiv.org/pdf/2105.01978)) | Design-time verification of adaptation logic; a knowledge base that could hold outcomes | No found work judges after the fact whether an adaptation was beneficial and holds the loop to it |

**Verdict for (b):** the idea of following up on what a filter rejected exists, published this year for trading, and the idea of a chained decision ledger for agent actions exists. No found work makes each automatic limit change an obligation with a horizon, a resolution by counterfactual replay, a named reason for staying open, and a bounded feedback into the policy. Phase 3 cites PRFS as the nearest precedent and the Agent Control Protocol for the ledger-before-action pattern.

### Pillar (c): which program classes bear the harm

No found work reports harm from an adaptive limiter by class of governed program. The four-fifths screen is standard in employment and credit (and already applied in this portfolio to the COMPAS data); its use on a limiter's decisions is, as far as this search goes, unreported. Adjacent: fairness and backpressure discussions in API rate limiting ([a practitioner newsletter](https://thebackenddevelopers.substack.com/p/adaptive-rate-limiting-in-distributed)) and vendor posts on false positives against legitimate bursts ([Security Boulevard](https://securityboulevard.com/2026/06/adaptive-rate-limiting-flash-sale-shoppers-vs-ddos-attacks/)), none with measurements that survive scrutiny (one implementation guide quotes a false-positive rate and a dollar figure with an unverifiable citation).

### Pillar (d): a benchmark of hard cases scored on net time

| Adjacent work | What it has | What it lacks |
|---|---|---|
| AgentTrust ([arXiv 2605.04785](https://arxiv.org/pdf/2605.04785)), AgentDojo, ToolEmu, AgentHarm, R-Judge, AgentBench, DreamGuard ([arXiv 2608.05695](https://arxiv.org/pdf/2608.05695)), a comparative guardrail evaluation ([arXiv 2604.24826](https://arxiv.org/abs/2604.24826), vendor-affiliated) | Benchmarks for agent safety: harmful tool calls, prompt injection, multi-step risk; runtime interceptors scored on verdict accuracy | Content and action safety, not resource governance; scored on verdict accuracy, not on time destroyed against time saved; no cold-start, headroom, patient-attacker or heavy-tail cases |
| Token Budgets: 63 budget-overrun incidents with an affine-typed Rust mitigation ([arXiv 2606.04056](https://arxiv.org/html/2606.04056)); Persistent Billable State: denial-of-wallet attacks and a host-side kernel ([arXiv 2609.28585](https://arxiv.org/pdf/2609.28585)); Agent Contracts ([arXiv 2601.08815](https://arxiv.org/abs/2601.08815)) with resource limits, temporal boundaries and success criteria; Agent Behavioral Contracts ([arXiv 2602.22302](https://arxiv.org/html/2602.22302.pdf)); AgentBound ([arXiv 2606.30970](https://arxiv.org/html/2606.30970v2)); "Scan the Skill, Govern the Action" ([arXiv 2609.12001](https://arxiv.org/pdf/2609.12001)) | **The nearest field.** Resource bounds on agents are an active 2026 topic: static budgets that cannot be exceeded, per-session and per-turn envelopes with a growth-anomaly monitor, formal contracts with time bounds, consequence control at runtime | Static or envelope enforcement, not adaptive tightening from observed behavior; evaluated on token reduction, variance and violations, not on net time or honest work destroyed; no adversarial roster of patient or inflating programs; no accountability for the enforcer's own mistakes |
| The 2025 AI Agent Index ([arXiv 2602.17753](https://arxiv.org/pdf/2602.17753)) | Sandboxing documented for 9 of 30 deployed agents | Documentation survey, not a benchmark |
| Counterfactual evaluation: long-horizon residual benchmarking ([arXiv 2607.27283](https://arxiv.org/html/2607.27283)); a queueing counterfactual task noting that policies which enter regimes the logs never reached are not estimable | The protocol-level counterfactual comparison the benchmark needs, and a warning that applies to phase 2: a limiter that kills what the logs never saw killed cannot be scored from logs alone | Not about limiters |

**Verdict for (d):** agent resource governance is an active field as of 2026, which is good for the program's relevance and bad for any claim of being alone in it. Every found evaluation scores enforcement on budgets kept or violations prevented. None scores on net time, none carries the hard cases that broke the first design, and none holds the enforcer accountable for honest work it destroyed.

### Overall verdict

- **Each pillar has adjacent prior art, and three of the four have close prior art published in 2026.** The program must cite PRFS, Agent Contracts, Token Budgets, Persistent Billable State, Agent Control Protocol, and the straggler cost models, and position against them.
- **No found work combines the four pillars**, and no found work states or measures the self-censoring loop of section 4, scores a limiter on net time, or reports its harm by program class.
- **The honest sentence** is therefore: "As far as a documented twelve-search check on 2026-10-15 can establish, no published work combines priced-in-time tightening, per-decision outcome obligations, class-level harm reporting and a net-time benchmark for adaptive limits on autonomous programs; the nearest precedents are PRFS for outcome follow-up and the 2026 agent resource-governance papers for static enforcement." Not "no one has this." The difference between those two sentences is the difference between a claim a referee can check and one they will enjoy refuting.
- **What would overturn this:** a paper or system found later that adapts a resource limit from observed behavior and reports the outcome of each adaptation. The check should be repeated before each phase's results commit, and the program document amended if the answer changes.
