# Adversarial Testbed: How the Loop Is Attacked, and What Held

**Status:** Twenty named threats, each tested, argued, or marked not covered, in `experimental/docs/THREAT_MODEL.md`. This is the map. Nothing here has run against a live adversary; the attackers are scripted agents in synthetic traces.

---

## The sharp question

Can an adversarial workload exploit the enforcement-feedback loop to cause incorrect automatic tightening, or to avoid deserved tightening? And can validation tell the two apart?

## Attacker classes

| Class | Can do | Covered |
|---|---|---|
| A1 observation-only | Controls the values and trap events the loop sees | Tested and measured |
| A2 API caller without operator rights | Calls governor methods, forges proposals | Tested |
| A3 holder of the server token | HTTP calls; with per-operator credentials, only as that operator | Tested when configured |
| A4 same-process code or write access to state | Edits memory or the log | Detection only |
| A5 network observer | Reads or alters HTTP traffic | Not covered (no TLS) |
| A6 malicious operator | Holds legitimate approval power | Not covered (logged, not constrained) |

## The attacks and the results

| Attack | How it is run | Result |
|---|---|---|
| Push a limit up through any path (mislabelled direction, rollback, store write, replayed grant) | Fuzzing with an observation-only attacker; API-misuse tests | Never raised a limit. Fixed defects T6, T7 along the way |
| Tightening as denial of service: send over-limit values until the floor | `experiments/dos_tightening.py`, fast and patient attackers | Reaches the floor in 16 values without the breaker; the breaker caps the loss at about 27% |
| Baseline poisoning: be present while the detector learns | `docs/BASELINE_POISONING.md` | 5% contamination cuts detection of a 2 sd shift from 100% to 13 to 36%. Opt-in trimmed fit survives light contamination only |
| Evasion under the detectors | Evolved attackers, `PHASE9_EXPERIMENT_RESULTS.md` | Partial detection; weak attacks evade |
| Spoofed expected-load pattern | `tests/test_phase13_integration.py` | Works unless pattern registration is an audited operator action (`strict_patterns=True`, off by default) |
| Low-and-slow: hang 0.8% of tasks, under the 1% rule | Trace agent `attacker-lowslow`, 20 seeds | Caught in 19 of 20 seeds anyway (local windows exceed 1%), but late: hung time saved 0 to 20% |
| Inflate first: slow completed tasks after the baseline is learned, then hang tasks | Trace roster `inflate_v1`, 6 agents across rosters, 20 seeds | The plain drift hold never tightens them (0 of 120 agent-seeds, T15 demonstrated). The excess-release rule tightens all 120, 3 to 7 slices late, saving 6 to 24% instead of 19 to 27% |
| Inflate from the start: poison the baseline | Trace agent `inflate-from-start` | Tightened normally (there is no drift to hold on) |
| Rewrite the audit log with every hash recomputed | `tests/test_audit_anchor.py` | Refused at startup against an anchor the process cannot rewrite; accepted if the attacker can also rewrite the anchor |
| A buggy signal source spends the budget | `tests/test_signal_interface.py` | Bounded at the breaker; the source cannot raise a limit or decide anything |
| Game the validation canary: complete the known canary tasks, hang the rest | Roster `canary_v1`, 20 seeds, round 8 | Predictable selection: DEGRADED on 140 of 140 reviews, restore request 20 of 20 seeds, tightening held. Keyed selection: 1 of 20, saving equal to an honest runaway's |
| Raise a limit below float resolution (10^18 to 10^18 + 50 compare equal as doubles) | Red team script, `experimental/docs/RED_TEAM.md` S1 | Succeeded before the fix: no grant needed, integrity audit green, survived restart. Fixed: exact rational comparison |
| Reset the breaker by acknowledging as any operator name, no registry configured | Red team S2 | Succeeded before the fix: 16 automatic tightenings on a boundary capped at 3. Fixed: operator actions refused without a registry unless the governor opts in for a demo |
| Flood 1,000 throwaway agent ids to lock a later agent out of governance | Red team S3 | Succeeded before the fix. Mitigated: quiet untightened agents are evicted; tightened ones never |
| Land on a victim's boundary with an ASCII id spelled like its hashed key | Red team S4 | Succeeded before the fix. Fixed: the prefix is reserved |

## What validation adds against gaming

The validation layer judges a tightening by a canary A/B comparison in time currency, blind to who the agent is. Against the inflate-first attackers it issues DEGRADED about half the time: in time terms the tightening really does burn more on their inflated-but-completing tasks than it saves on their hangs. That is the right number and the wrong intent. A person receives a restore request and has to refuse it. Validation therefore does not distinguish adversarially induced tightening from beneficial tightening by itself; it distinguishes costly tightening from cheap tightening, and leaves the intent to the operator.

## Not tested

- Coordinated attackers spreading violations across agents or boundaries so no single pattern forms.
- The decoy attack from the critique (draw tightening to cheap boundary A, then attack B): per-agent, per-limit boundaries and per-boundary breakers mean A's tightening does not spend B's budget, so the attack as described does not apply, but a variant across many throwaway agent ids has not been tried (the agent cap of 1,000 is the only guard).
- ~~Attacks on the canary itself (an agent that behaves on every tenth task).~~ Tested in round 8: with a predictable canary the attacker is gamed on every review and a restore request is filed on every seed; with a keyed selection the same attacker is treated like an honest runaway. Condition: the key is secret from the agent and the kernel does not reveal a task's deadline. Still untested: per-task probing of membership by observing when a task is killed.
- Anything against real traffic.
