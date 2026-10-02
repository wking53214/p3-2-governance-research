# Architectural Overview: Enforcement Telemetry Governance

**This document:** High-level architecture without overclaiming  
**What it covers:** What the system actually does  
**What it doesn't claim:** That it works, that violations always point to correct boundaries, or that tightening improves security

---

## The Core Loop

```
untrusted execution
    ↓
boundary violation
    ↓
immutable trap event
    ↓
aggregation into patterns
    ↓
interpretation layer (?)
    ↓
proposed constraint change
    ↓
validation layer (?)
    ↓
automated tightening OR human-approved loosening
    ↓
next execution
```

The feedback loop means: earlier executions' enforcement pressure informs later executions' constraints.

---

## Key Components

### Execution Contexts
Untrusted code runs in contexts. A context is a call chain (API entry → handler → nested calls).

Contexts are tracked to ask: "Have we seen this execution pattern before? Did it violate boundaries last time?"

### Boundaries
A boundary is a constraint: timeout, resource limit, operation count, coherence gate, etc.

Boundaries are mostly static (set by architects) but can change based on observed pressure.

### Trap Events
Every enforcement action creates an immutable record: when, which context, which boundary, what happened, how long recovery took.

This is the system's source of truth about enforcement.

### Trap Patterns
Pre-computed summaries of trap_events: "Context family X violated boundary Y how many times in what time window?"

Updated periodically (streaming or batch), these summaries feed into live decisions without scanning raw events.

### Interpretation Layer ⚠️
**Currently missing from the architecture.**

This layer asks: "What does this violation pattern mean?"

Without it, the system treats all violations equally and can be gamed by adversarial patterns designed to look meaningful but that point to wrong boundaries.

### Validation Layer ⚠️
**Currently missing from the architecture.**

This layer asks: "Did tightening this boundary actually improve security?"

Without it, the system may tighten boundaries that reduce violations but don't actually prevent attacks.

---

## Safety Properties

### What The Architecture Provides

1. **Monotonic tightening:** Observed pressure automatically tightens. No human approval needed for tightening.

2. **Gated loosening:** Absence of violations doesn't automatically loosen. Loosening requires human approval.

3. **Asymmetric authority:** Creates a one-way ratchet. The system can't be trained into weakness by simply avoiding violations.

4. **Immutable audit trail:** All enforcement actions are recorded permanently. You can replay decisions, ask "why did the system do that?", and trace back to source events.

5. **Queryable telemetry:** Enforcement events are structured data, not logs. You can ask: "Which contexts violate which boundaries? What's the pattern? Is it getting better or worse?"

### What The Architecture Does NOT Provide

1. **Proof that tightening improves security:** The system observes violations. Whether tightening in response improves security is a separate question that requires validation.

2. **Immunity to specification gaming:** Adversarial code can deliberately generate patterns designed to look meaningful but point to wrong boundaries.

3. **Elimination of human oversight:** Humans are still in the loop. They approve loosening, adjudicate constraint accumulation, validate outcomes. The architecture relocates human effort but doesn't eliminate it.

4. **Autonomous learning of correct boundaries:** The system learns where code violates constraints. Learning what the constraints should be requires external input (domain expertise, outcome validation, etc.).

---

## How Data Flows

### Live Path (Sub-Millisecond)

Code executes → preemption checks trap_patterns → reads "this context violated this boundary N times in 24h" → makes enforcement decision.

One row lookup. No joins. Microseconds.

### Historical Path (Batch, Seconds OK)

Periodically, background job scans trap_events → groups by (context, boundary) → counts violations → updates trap_patterns.

Later, analyst queries trap_events directly to understand patterns in detail.

### The Window

Between when a violation occurs and when it influences future decisions, there's a window:

```
violation → event → aggregation → interpretation → decision → actuation
```

Even with real-time streaming, this window exists. Attacker can change behavior within it.

---

## Data Model (Simplified)

### trap_events (Immutable Log)

```
event_id, timestamp, context_id, boundary_id, deadline_id
outcome (TRAPPED|ALLOWED|DEFERRED), recovery_latency_ms
metadata_json
```

**Key property:** Append-only. Never updated or deleted. Source of truth.

### execution_contexts (Live Cache)

```
context_id, domain, call_stack_hash, entry_point, parent_context_id
last_activity_timestamp, metadata_json
```

**Key property:** Tracks patterns so you don't store 1M unique contexts, you store 10K unique call stack signatures.

### boundaries (Reference)

```
boundary_id, type (TIMEOUT|RESOURCE|PREEMPT|COHERENCE)
layer, threshold, severity, created_timestamp
effectiveness_score ⚠️ (UNDEFINED - see critique)
```

**Key property:** Small table. Rarely changes. Reference data.

### trap_patterns (Materialized Summary)

```
context_hash, boundary_id
trap_count_1h, trap_count_24h, trap_count_7d
mean_recovery_latency_ms
effectiveness_signal ⚠️ (UNDEFINED)
digest_timestamp (when last updated)
```

**Key property:** Pre-computed answers to "how many violations?" Live code reads this, not trap_events.

---

## What Needs To Work

For the architecture to actually function:

1. **Detection must work:** Can we identify real patterns in noise?

2. **Interpretation must work:** Given a pattern, can we decide what boundary to adjust?

3. **Validation must work:** Can we prove the adjustment improves security?

If any one of these fails, the architecture either breaks or needs redesign.

---

## Comparison to Existing Approaches

| Approach | Governance Model | Scaling | Human Effort | Proof Required |
|----------|---|---|---|---|
| Static boundaries | Set once, never change | OK | Low (setup) | Specification (trust experts) |
| Red-teaming | Find holes, patch | Poor | High (continuous) | Testing (vulnerability found) |
| RLHF | Supervised learning | Moderate | High (labeling) | Training data quality |
| **This proposal** | **Enforcement telemetry** | **Better** | **Medium (approval)** | **Validation layer** |

The proposal aims to shift human effort from "watch every execution, patch on discovery" to "periodically review accumulated constraints, validate they work."

That's a different trade-off, not elimination of human involvement.

---

## Known Gaps

### 1. The Interpretation Layer Is Unspecified

The paper jumps from "violation occurred" to "tighten boundary." 

In reality, you need: "This violation occurred for reason X. Should we tighten boundary Y?"

Those are different questions.

### 2. Validation Is Unspecified

No mechanism for proving that boundary tightening actually improves security.

Without this, the system may optimize for reducing violations while ignoring whether reduced violations correlate with improved security.

### 3. Constraint Accumulation Has No Clear Resolution

Asymmetric authority means constraints accumulate forever. Human review stops runaway accumulation but creates periodic human governance burden.

This is a feature (safety) but it's not "without human oversight."

### 4. The Materialization Checkpoint Is Unsafe

Using MAX(digest_timestamp) to track progress through trap_events doesn't work when different rows have different timestamps. See CRITIQUE.md for details.

### 5. Effectiveness Score Is Undefined

The schema includes `effectiveness_score` but never defines what "effectiveness" means. Is it security? Speed? Low violation count? Something else?

---

## Next Steps

Before deploying or building:

1. **Specify the interpretation layer:** How do you decide which boundary to adjust given a violation pattern?

2. **Design the validation layer:** How do you prove tightening improves security (not just reduces violations)?

3. **Run adversarial tests:** Can malicious code deliberately generate misleading patterns? Can validation layer detect this?

4. **Implement on a real workload:** Deploy to ML models, autonomous agents, or trading algorithms. Measure whether it actually reduces security incidents.

If all three pass, the architecture has merit. If one fails, you learn what needs to change.

---

**Status:** Architectural proposal, needs research validation before deployment.
