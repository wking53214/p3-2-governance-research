# Three Research Questions

This architecture proposal breaks down into three separable research problems. Each must be solved for the overall system to work.

---

## A. Detection: Can We Reliably Identify Patterns in Enforcement Events?

**The Question:**
Given a stream of enforcement events (boundary violations, timeouts, preemptions), can we extract meaningful patterns that tell us something true about the relationship between workloads and constraints?

**What makes this hard:**
- **False positives:** Detecting a pattern that doesn't exist. Example: "Context X is problematic" when actually the execution environment was temporarily degraded.
- **False negatives:** Missing a pattern that does exist. Example: An attacker violating boundaries in a pattern designed to look like noise.
- **Cardinality:** With millions of execution contexts, pattern detection algorithms can drown in false signals.
- **Temporal dynamics:** A pattern valid on Monday might not be valid on Tuesday (load changes, code updates, etc.).

**Success looks like:**
- Detection algorithm flags pattern P
- Pattern P accurately describes an actual constraint/workload relationship
- Confidence score correctly reflects likelihood of false positive
- Pattern doesn't change every time a new execution occurs (stability)

**Research approach:**
- Run detection on known-good and known-bad workloads
- Measure false positive rate and false negative rate separately
- Vary cardinality and complexity of event streams
- Test against adversarial workloads designed to generate misleading patterns

---

## B. Adaptation: Can We Safely Turn Patterns Into Constraint Changes?

**The Question:**
Given a detected pattern (e.g., "Context family X violates boundary Y 10 times per day"), can we propose a boundary change that is safe and effective?

**What makes this hard:**
- **Ambiguity:** A pattern tells us *that* something happened, not *why*. Context X violates the 100ms deadline could mean:
  - Deadline is too tight (should be 120ms)
  - Context needs optimization (should run in 80ms)
  - Boundary is measuring the wrong thing
  - Environment is degraded
  - Code is malicious and probing limits
  
- **Gaming:** Adversarial code can generate patterns designed to trick the adaptation layer into making the wrong choice. Example: Deliberately violate cheap boundary A (fast recovery) to cause system to tighten A, then attack expensive boundary B while A is defended.

- **Cascading failures:** Tightening boundary A can cause failures on boundary B, creating false patterns about B.

- **One-way safety ratchet:** Asymmetric authority means tightening is automatic but loosening requires approval. This prevents undoing good tightening but also means bad tightening accumulates.

**Success looks like:**
- Adaptation algorithm proposes change C
- Change C is safe (doesn't cause functional degradation or safety violations)
- Change C reduces future violations of the same pattern by >50%
- Adversarial workload cannot cause an unsafe change to be proposed

**Research approach:**
- Design an adversarial testbed where attacker sees enforcement events and tries to generate patterns that cause bad boundary changes
- For each proposed boundary change, measure expected functional impact and expected security impact
- Test on real workloads (ML models, autonomous agents, algorithmic trading, etc.)

---

## C. Validation: Can We Prove Tightening Improves Security?

**The Question:**
When we tighten boundary Y from 100ms to 90ms, how do we know we're making the system more secure rather than just making it faster?

**Why this is hard:**
- **No oracle:** There's no ground truth about what a "secure boundary" looks like until you run the system and something happens (or doesn't).
- **Long feedback loop:** Security improvements might take weeks or months to become visible.
- **Correlation vs. causation:** Violations might drop after tightening because the attacker changed tactics, not because the boundary now works.
- **Functional cost:** Tightening boundaries that are meant to be loose can cause legitimate code to fail.
- **Silent failures:** Code might adapt to tighter boundaries in ways that are faster but less correct.

**The proposal in the current architecture:**
- Independent validation layer checks outcomes separately from trap_events
- For lending: outcome audit asks "did the recommendation make sense?"
- For authorization: downstream monitoring asks "did the user do anything malicious?"
- For vehicles: safety validation asks "did preemption prevent an accident?"

**The problem:** 
This requires defining "security" per domain and building a validation system that's itself trustworthy. That's a whole other research program.

**Success looks like:**
- Tightening boundary Y from 100ms to 90ms
- Measure violation rate (does drop)
- Measure security incidents downstream (do they drop more than violation rate?)
- Measure functional performance (does it stay acceptable?)
- Confirm causation via A/B test or counterfactual

**Research approach:**
- Build domain-specific outcome audits (lending: recommendation quality, hiring: hiring decisions, vehicles: accident rate)
- Use these as ground truth for validation
- Run controlled experiments: tighten boundary, measure outcome, compare to control group
- Test that validation layer itself can't be gamed

---

## How They Interact

Detection → Adaptation → Validation → Loop

```
enforcement events
     ↓
[DETECTION]
     ↓
meaningful patterns
     ↓
[ADAPTATION]  
     ↓
proposed constraint change
     ↓
[VALIDATION]
     ↓
does this improve security?
     ↓
NO → discard, try different pattern
YES → apply change, observe next cycle
```

---

## Where The Current Proposal Fails

The white paper addresses Detection and Adaptation with engineering detail. It mentions Validation but doesn't establish a mechanism for it.

This means the proposal claims "the system hardens itself" without proving it actually hardens toward *security* (rather than toward faster execution, or toward patterns an adversary deliberately generated).

---

## Success Criteria

All three questions must be answered affirmatively for the overall architecture to work:

- **A (Detection) succeeds:** We can identify patterns reliably
- **B (Adaptation) succeeds:** We can propose changes safely  
- **C (Validation) succeeds:** We can prove those changes improve security

If any one fails, the architecture either breaks or needs to be redesigned around the failure.

---

## Research Status

**Detection:** Plausible, standard engineering problems (stream aggregation, cardinality management, statistical tests)

**Adaptation:** Plausible with interpretation layer, but interpretation layer itself is unspecified

**Validation:** Unspecified. This is the hard part.

---

**Current assessment:** 

The architecture is interesting as a proposal. It can work if all three research questions are solved. The white paper skips C, which is why it overclaims.
