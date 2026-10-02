# Self-Hardening Governance Architecture for Untrusted Code Execution

**Author:** William N. King  
**Collaborative Research:** Claude Fable 5.1, Claude Haiku 4.5  
**Date:** October 2, 2026

---

## Executive Summary

Current approaches to governing untrusted code execution require either constant human oversight, reactive boundary adjustments, or expensive red-teaming cycles. We present a novel architecture where enforcement events themselves become learning signals, enabling systems to autonomously and continuously harden boundaries against untrusted code without external energy injection.

The core insight: every time untrusted code violates a boundary, that violation is not merely a failure to contain—it is data. When captured in a queryable relational schema and fed through a materialized learning loop, these violations become the primary mechanism by which the system tightens the very boundaries that should have contained them. The result is a self-governing execution environment where the act of preemption immediately informs the next preemption, creating a perpetual tightening flywheel.

This architecture addresses a fundamental problem in autonomous systems: anti-isolation. Systems must not hide in their own conceptual universes. They must be forced to visibility about what they enforce, when they enforce it, and why. A relational schema connecting execution contexts, boundary violations, deadline windows, and recovery patterns creates that visibility and makes autonomous learning possible.

The novelty is not in any single piece—hot/cold data tiers, materialized views, stream processing, and adversarial governance all exist in literature. The novelty is the unified application: using a governance kernel to enforce boundaries on untrusted execution, capturing enforcement events in an immutable audit trail, materializing patterns from that trail, and feeding those patterns back into boundary tightening decisions, all without human intervention in the feedback loop itself.

We examine ten critical failure modes that could subvert this design, identify which are architectural (solvable through design changes) and which are policy-level (requiring human gates and validation), and demonstrate that the system can be hardened against adversarial code that attempts to game the learning loop.

Real-world implications span AI safety (untrusted model deployment), autonomous systems (vehicle and robot governance), financial systems (algorithmic containment), and enterprise governance (scale without scaling human oversight).

---

## 1. The Problem Statement

### Current Limitations of Execution Governance

Governing untrusted code execution at scale remains unsolved. Organizations deploying third-party models, plugins, autonomous agents, or adversarial code face one of three paths, all expensive:

**Static Boundaries:** Define resource limits, timeout windows, and behavioral constraints once, apply them uniformly forever. This is simple and safe but inflexible. A boundary that was appropriate for code A may be too loose for code B or wasteful for code C. Static governance scales to deployment but not to effectiveness.

**Reactive Red-Teaming:** Find vulnerabilities through human testing, patch boundaries, repeat. This is responsive but expensive. Red-teaming talent is scarce. Vulnerabilities found are only those the team thinks to test. Discovery is slow relative to attack innovation.

**Supervised Learning (RLHF):** Collect human feedback on model behavior, retrain the model or its constraints. This requires human labels, breaks between update cycles, and redeployment. The system cannot adapt to new patterns faster than the training cycle allows.

**Continuous Human Monitoring:** Operators watch dashboards, approve boundary changes, intervene when code misbehaves. This works but doesn't scale. Adding 10x more code requires 10x more human oversight.

### The Cost of Scale

Suppose you deploy 1,000 untrusted models in a financial institution. Each model is different: different architectures, different training data, different attack surface. Static boundaries will contain some and fail on others. Red-teaming 1,000 models is prohibitively expensive. Supervised learning would require thousands of human labels per model. Continuous monitoring would require a team of 50+ operators watching for anomalies.

Yet if a single model slips governance and makes a bad recommendation, the cost is measured in millions of dollars and regulatory penalties. Governance cannot be relaxed for scale.

### Why Existing Approaches Fail for Autonomous Deployment

The fundamental problem: current approaches treat governance as *external to the governed system*. Humans define boundaries. Humans monitor violations. Humans decide to tighten. The system being governed is passive—it receives constraints but does not participate in refining them.

This works for static, small-scale deployments. It fails for autonomous, large-scale deployment because:

1. **Feedback loops are slow.** Humans detect a problem, discuss it, decide on a fix, approve the change, deploy it. Days to weeks. Code running at 1,000 req/sec can try thousands of exploits in that time.

2. **Humans don't scale.** Monitoring 1,000 systems requires 1,000x human attention. The cost of governance exceeds the value of the code.

3. **Patterns are invisible.** A single violation is an anomaly. A pattern of violations is a signal. Humans can't see patterns across 1,000 systems in real time; they see dashboards, which compress and abstract away the data that would reveal patterns.

4. **Governance becomes static.** Once deployed, boundaries rarely change. They get tighter only if there's a visible crisis. They never tighten based on patterns that don't trigger alarms.

### The Anti-Isolation Principle

STACK's foundational principle: systems must not trap themselves in their own conceptual universes. A system that cannot see what it's enforcing, when it's enforcing it, or why cannot learn from enforcement. It hides inside its own model.

Anti-isolation means: make enforcement visible. Capture it. Query it. Use it. Let the system observe its own enforcement and use those observations to refine enforcement.

A system that can do this is not trapped. It is forced to transparency about what it's doing and why.

---

## 2. The Architecture: Relational Foundation for Autonomous Governance

### Core Concepts

The system must answer two classes of queries simultaneously:

1. **Live queries (sub-millisecond):** "Given that this untrusted code is executing in context X and just hit boundary Y, should I preempt it now?" This query happens during execution. Latency is critical. One millisecond of latency on 10,000 concurrent executions is unacceptable.

2. **Historical queries (batch, seconds OK):** "Over the last 7 days, which execution contexts trap repeatedly at boundary type Z? What's the pattern? Should boundaries change?" This query happens offline. Latency is irrelevant; thoroughness is critical.

A naive approach would separate these completely: one database for live decisions, one for analysis. This creates data consistency problems and operational burden.

Instead: a single schema, partitioned by time and optimized with multiple tiers. Recent data lives in a hot tier optimized for live queries. Older data moves to a cold tier optimized for analytical scanning. Both are queryable; the query optimizer chooses which tier to scan.

### Entities and Relationships

#### trap_events (Immutable Audit Log)

Every time untrusted code is preempted—every boundary violation, every timeout, every resource exhaustion—one row is appended to trap_events. This log is immutable. Rows are never updated or deleted. This is the system's source of truth.

**Key columns:**
- `event_id` (PK): Unique identifier, monotonically increasing with time
- `timestamp`: Exact moment preemption triggered (microsecond precision)
- `execution_context_id` (FK): Which execution context was running
- `boundary_id` (FK): Which boundary caught the violation
- `deadline_window_id` (FK): Which deadline constraint was active
- `outcome`: TRAPPED | ALLOWED | DEFERRED | ESCALATED
- `trap_depth`: Stack depth when caught (for pattern analysis)
- `recovery_action`: What the preemption handler did (unwind, suspend, kill)
- `recovery_latency_ms`: Time from preemption signal to full recovery
- `domain_id`: Which domain (LENDING, HIRING, AUTHORIZATION, etc.)
- `tenant_id`: Multi-tenant isolation
- `metadata_json`: Flexible attributes (call stack hash, resource state, etc.)

**Why this structure:**
- Immutable: no UPDATE/DELETE, only INSERT. Guarantees audit trail integrity.
- Denormalized slightly (domain_id, tenant_id): for fast partitioning and access control.
- Rich metadata: analytics later might reveal patterns in stack depth, recovery latency, etc.

#### execution_contexts (Live Cache of Running Code)

Untrusted code runs in execution contexts. A context is a call chain: API entry point → handler → nested calls. Contexts have history (we've seen this call pattern before) and metadata (what it's trying to do).

**Key columns:**
- `context_id` (PK)
- `domain`: LENDING | HIRING | AUTHORIZATION (what business purpose)
- `call_stack_hash`: 64-bit hash of the call stack (for grouping similar contexts)
- `entry_point`: Which API entry point initiated this (e.g., /approve_loan)
- `parent_context_id`: For nested untrusted calls (context A calls context B)
- `created_timestamp`: When this context started
- `last_activity_timestamp`: Last time code in this context ran (hot for time-decay)
- `isolation_group`: Which isolation domain (for access control)
- `metadata_json`: Flexible attributes

**Why this structure:**
- Call stack hashing: don't store full stacks (cardinality explosion); hash them. Similar stacks hash to the same value.
- Hierarchy: nested contexts inherit properties from parent.
- Time decay: recent contexts are "hotter"; old contexts can be archived.

#### boundaries (Reference Data)

A boundary is a rule: "don't do X" or "stop after Y time" or "don't use more than Z resources." Boundaries are mostly static, added by architects, occasionally updated by policy.

**Key columns:**
- `boundary_id` (PK)
- `boundary_type`: PREEMPT | TIMEOUT | RESOURCE | COHERENCE | CUSTOM
- `layer`: KERNEL | RUNTIME | SUPERVISION | AUDIT (where in the stack it operates)
- `threshold`: What triggers it (deadline in ms, memory in GB, operation count, etc.)
- `severity`: CRITICAL | WARNING | INFO
- `created_timestamp`
- `effectiveness_score`: Updated daily from trap data (0.0 to 1.0)
- `is_active`: Can be disabled without deletion

**Why this structure:**
- Reference data: small table (50-200 rows), rarely changes.
- Effectiveness score: allows querying "which boundaries are actually helping?"
- Layered: boundaries at different levels (kernel preemption, runtime limits, audit gates).

#### deadline_windows (Temporal Constraints)

Untrusted code runs under time constraints. A deadline window specifies: "code running under this deadline can execute for at most N milliseconds." Windows are created on demand, exist for the duration of execution, then close.

**Key columns:**
- `window_id` (PK)
- `start_timestamp`: When this deadline started
- `end_timestamp`: When it expires
- `deadline_ms`: Maximum execution duration allowed
- `priority_tier`: REALTIME | NORMAL | BATCH (SLA)
- `domain_id`: Which domain this deadline applies to
- `created_by`: What process created this (for audit)

**Why this structure:**
- Temporal: windows are created and destroyed; the table grows/shrinks.
- Priority tiers: realtime code gets stricter deadlines than batch code.
- Partition by domain: can ask "do LENDING deadlines have more violations?"

#### trap_patterns (Materialized View, Pre-Computed Summaries)

This is the key to live query speed. Instead of scanning millions of trap_events every time you need a live decision, trap_patterns pre-computes summaries: "execution context X, boundary Y, how many traps in the last 24 hours?"

Updated asynchronously (every 5 minutes), this table holds the learning signal in a form that live queries can consume without scanning raw events.

**Key columns:**
- `pattern_id` (PK)
- `context_hash`: Which execution context (64-bit hash)
- `boundary_id` (FK): Which boundary
- `trap_count_1h`: Traps in last hour
- `trap_count_24h`: Traps in last 24 hours
- `trap_count_7d`: Traps in last 7 days
- `last_trap_timestamp`: When the last trap occurred
- `mean_recovery_latency_ms`: Average recovery time
- `effectiveness_signal`: Trending up (boundary getting better) or down (worse)
- `digest_timestamp`: When this row was last updated

**Why this structure:**
- Materialized: pre-computed, not derived on the fly.
- Granular time windows: 1h, 24h, 7d allow queries at different scales.
- Effectiveness signal: system can ask "is this boundary actually helping?"
- One row per (context, boundary) pair: fast lookup from live code.

### Entity Relationships (ER Diagram as Text)

```
execution_contexts
    |
    ├─→ (1:M) trap_events
    |        |
    |        ├─→ (M:1) boundaries
    |        ├─→ (M:1) deadline_windows
    |        └─→ [joins reveal patterns]
    |
    └─→ trap_patterns
         └─→ [fed back into live decisions]
```

**Normalization rationale:**
- trap_events is normalized: references context_id, boundary_id, window_id (FKs). This prevents duplication and maintains integrity.
- trap_patterns is denormalized: copies boundary_type and context_hash (not FKs). This trades storage for speed; live queries can answer "which boundary type?" without joining boundaries.
- execution_contexts is normalized for hierarchy (parent_context_id is FK).
- Partitioning: all tables partition by (tenant_id, domain_id) for multi-tenant isolation and fast pruning.

### Hot/Cold Data Tiering

**Hot tier (last 48 hours):**
- All trap_events from the last 48 hours
- All execution_contexts with last_activity_timestamp > now() - 2 days
- All trap_patterns (tiny, fits in memory)
- Indexes: aggressive, multiple indexes per table
- Location: RAM or fast SSD
- Query optimizer: scans hot tier first, indexes prevent full scans

**Warm tier (2-30 days):**
- trap_events 2-30 days old
- execution_contexts inactive for >2 days, <30 days
- Fewer indexes; analytical queries allowed but slower
- Location: SSD

**Cold tier (30+ days):**
- trap_events >30 days old
- Minimal indexes (metadata only)
- Location: Columnar database (ClickHouse) or S3 archive
- Queries: rare, batch-only

**Automatic aging:** Scheduled job at 00:05 each day promotes yesterday's hot data to warm, and 30-day-old warm data to cold.

---

## 3. The Flywheel Mechanism: How Enforcement Becomes Learning

### The Loop

1. **Untrusted code executes** in context X, hits boundary Y at time T.
2. **Preemption signal fires.** Signal handler sets a flag.
3. **Next safe boundary checkpoint** catches the flag. Exception thrown. Context unwinds. Resources cleaned up.
4. **trap_events row inserted:** timestamp=T, context_id=X, boundary_id=Y, outcome=TRAPPED, recovery_latency_ms=M.
5. **Async materialization job** (runs every 5 minutes): scans trap_events since last update. Aggregates by (context_hash, boundary_id). Updates trap_patterns.
6. **Next execution in context X** reads trap_patterns: "this context has trapped at boundary Y 7 times in 24 hours. Effectiveness signal is trending up (boundary is working)."
7. **Live decision:** System tightens boundary Y for context X. Deadline reduced from 100ms to 50ms. Or resource limit reduced.
8. **Next execution in context X:** Hits the tighter boundary earlier. Same violation now caught sooner. Stack didn't unwind as far. Recovery faster.
9. **trap_events row inserted:** same context, same boundary, earlier trap, faster recovery. Signal captured.
10. **Loop repeats.** Each cycle tightens the boundary. System learns where the real constraint is.

### Why This Works Without External Energy

Every step creates friction: code running, boundaries checking, preemptions firing, logs being written. In a reactive system, that friction is waste. In this system, friction is fuel.

The system doesn't require external feedback ("Hey, make boundary Y tighter"). It extracts the feedback from the cost of containment itself. The more often code violates a boundary, the tighter that boundary becomes. The system hardens in the direction it needs to harden because that's where the friction is highest.

This is anti-isolation at work: the system cannot hide in its model of how boundaries should work. It must observe how they actually work and adjust.

### Materialization Job Design

The job runs every 5 minutes. Pseudocode:

```
FUNCTION materialize_trap_patterns():
    since_timestamp = SELECT MAX(digest_timestamp) FROM trap_patterns
    
    new_events = SELECT * FROM trap_events 
                 WHERE timestamp > since_timestamp
    
    FOR EACH (context_hash, boundary_id) group in new_events:
        counts = COUNT(*) GROUP BY timebucket
        recoveries = AVG(recovery_latency_ms)
        signal = COMPUTE_TREND(counts_7d, counts_24h, counts_1h)
        
        UPSERT INTO trap_patterns
        SET trap_count_1h = counts[1h],
            trap_count_24h = counts[24h],
            trap_count_7d = counts[7d],
            mean_recovery_latency_ms = recoveries,
            effectiveness_signal = signal,
            digest_timestamp = now()
        WHERE context_hash = context_hash
          AND boundary_id = boundary_id
    
    UPDATE trap_patterns
    SET trap_count_1h = 0
    WHERE last_trap_timestamp < now() - 1 hour
        AND trap_count_1h > 0
```

**Key design:**
- Idempotent: can re-run without corruption
- Fault-tolerant: if job crashes mid-run, next run picks up where it left off (tracked via MAX(digest_timestamp))
- Efficiency: only updates rows where counts changed

### How trap_patterns Drives Live Decisions

Live query when context X is preempted at boundary Y:

```sql
SELECT trap_count_24h, effectiveness_signal
FROM trap_patterns
WHERE context_hash = hash(call_stack_of_X)
  AND boundary_id = Y
  AND digest_timestamp > now() - 2 hours
LIMIT 1
```

One row lookup. Microseconds.

**Live decision logic:**
```
IF trap_patterns.trap_count_24h > 5:
    IF effectiveness_signal TRENDING_DOWN:
        "Boundary is getting worse; don't loosen"
    ELSE:
        "Boundary working; we can relax it slightly"
        ADJUST boundary_threshold DOWN 10%
ELSE IF trap_patterns.trap_count_24h > 1:
    "Pattern emerging; watch it"
ELSE:
    "Isolated event; probably OK"
```

The system doesn't automatically loosen boundaries (that requires approval). It *does* automatically tighten.

---

## 4. Red-Team Analysis: Failure Modes and Mitigations

We identified ten critical failure modes. For each, the question is: can an adversary exploit it, and what's the fix?

### 1. Adversarial Batch Cycle Timing

**Attack:** Code learns the materialization cycle is every 5 minutes. Violates boundary at 12:00:00. Doesn't violate again until 12:05:01 (after digest_timestamp has advanced). trap_patterns never accumulates a pattern. Code violates indefinitely without being flagged as repetitive.

**Why it works:** Batch cycles are predictable. Code can synchronize violations to stay under detection thresholds.

**Mitigation (Architecture):** Replace batch materialization with real-time stream processing (Kafka → Flink or custom stream consumer). Every trap event immediately updates trap_patterns counters. Latency: milliseconds, not minutes.

**Cost:** Higher computational overhead. Must handle concurrent updates safely (atomic counters or CRDTs).

**Residual risk:** Stream processor itself could be compromised. Requires monitoring that the processor is running and emitting updates.

---

### 2. False-Positive Cascade

**Attack:** Legitimate code hits boundary A (rare edge case, slow recovery). System learns: "boundary A is ineffective, slow recovery means loosen it." Boundary A relaxed. Malicious code now unguarded.

**Why it works:** System assumes violation means boundary is too tight. Sometimes violations are legitimate.

**Mitigation (Policy):** Asymmetric governance. System tightens freely. System can only *loosen* boundaries with human approval. A human reviews: "trap_count is 1, recovery was slow, but code looks legitimate. Approve loosening? Yes/No."

**Cost:** Human approval gate required. Scales reasonably (one approval per boundary loosening, not per trap).

**Residual risk:** Humans might approve loosening a boundary that code will then attack. Mitigated by requiring that loosening must correlate with evidence that original boundary was causing problems, not just that code violated it.

---

### 3. Materialization Anomaly (Corrupted Learning Signal)

**Attack:** Two async materialization jobs run concurrently (clock skew, restart). Both UPSERT trap_patterns. Row ends up with counts from two different time windows mixed together. Live decisions read garbage.

**Why it works:** UPSERT with no serialization is not atomic across concurrent writers.

**Mitigation (Architecture):** Materialization is idempotent and single-writer. Only one job instance runs at a time (leader election or exclusive lock). If job crashes, next job replays from last successful digest_timestamp.

**Cost:** Requires coordinated scheduling (Temporal, Airflow, or custom).

**Residual risk:** Lock contention could delay materialization. Mitigated by keeping job runtime short (<100ms even on large event volumes).

---

### 4. Cardinality Explosion

**Attack:** 1,000,000 unique execution contexts × 50 boundaries = 50,000,000 potential rows in trap_patterns. Materialization job has to process billions of events per cycle. Job takes 30 minutes. Lag grows to 30+ minutes. System is blind.

**Why it works:** Cardinality explosion when grouping by fine-grained identity (context_id instead of context_hash).

**Mitigation (Architecture):** Group by call_stack_hash (64-bit), not context_id. Similar execution patterns hash to same value. Cardinality: ~10K distinct stacks, not 1M. For high-trap-rate stacks, promote to full context tracking.

**Cost:** Lose per-context precision. One trap_pattern row covers 100 similar contexts.

**Residual risk:** Attacker with two different legitimate stacks might hash to same bucket (unlikely with 64-bit hash) and interfere with each other's patterns. Mitigated by bucket-splitting when collision rate exceeds threshold.

---

### 5. Observation-Based Oracle

**Attack:** Malicious code reads trap_events (if it has query access). Learns: "boundary A is triggered 100 times/day and takes 2ms to recover. Boundary B is triggered 5 times/day and takes 50ms to recover. Boundary B is the real constraint." Focuses attack on B. System sees: "A is never tripped, must be redundant." Loosens A. Code ignores A.

**Why it works:** If code can read the audit trail, it can reverse-engineer system behavior.

**Mitigation (Architecture):** Separate access tiers. Code can read trap_events (audit trail) if query access is required for domain reasons. But code cannot read trap_patterns (learning summaries). If code must have query access to trap_events, encrypt the trap_patterns tier completely.

**Alternative (Policy):** Code has no query access to any trap data. Only enforcement layer reads trap_events and trap_patterns. Code cannot learn anything about its own traps.

**Cost:** First alternative: encryption overhead, separate access controls. Second alternative: loss of flexibility if code needs to introspect its own behavior.

**Residual risk:** If code can access trap_patterns anyway (compromise, misconfiguration), it gets the full oracle. Mitigated by auditing query access.

---

### 6. Slow Drift Problem

**Attack:** Code deliberately stays just-within boundaries. Over weeks, system observes "no violations, everything working." Gradually loosens boundaries. Code drifts to just-outside. Weeks later, code stops respecting boundaries entirely.

**Why it works:** Absence of violations is interpreted as "boundaries OK, can relax." Code exploits gradual drift.

**Mitigation (Policy):** Monotonic tightening by default. System can tighten boundaries freely. System can only loosen with explicit approval, tied to evidence that boundary was causing problems, not just that it's unused.

**Cost:** Boundaries never relax automatically. If a boundary was set too tight, human approval required to fix it.

**Residual risk:** Over years, many boundaries might accumulate, some tight and redundant. System becomes over-constrained. Mitigated by periodic review (quarterly: audit all boundaries, ask "is this still necessary?").

---

### 7. Ground-Truth Collapse

**Attack:** System measures success by "no violations." But code violates *intentions*, not boundaries. Code respects all deadline limits but makes incoherent decisions (lending recommendation violates underwriting principles, but no boundary was set for underwriting). trap_events shows no violations. System learns: "all boundaries working." Sentinel doesn't catch incoherence.

**Why it works:** Boundaries are reactive (enforce limits). Intentions are proactive (ensure correctness). System only learns from boundary violations, not intention violations.

**Mitigation (Policy):** Independent coherence validation. A separate system validates outcomes independently of trap_events. "Did the lending recommendation make sense? Was it understandable?" If outcome is incoherent, flag it separately. Don't use trap_events as the only signal.

**Cost:** Requires defining "coherence" independently. For lending, might be expert review or outcome audits.

**Residual risk:** Coherence validation might be weak. Code that is technically coherent but business-wise bad won't be caught. Mitigated by domain expertise in coherence rules.

---

### 8. Insufficient-Data Trap

**Attack:** Code runs rarely (10 executions/day, at 3 AM). Gets trapped once. trap_patterns shows trap_count=1. System doesn't tighten (threshold is 5 traps for a pattern). Code violates for 5 days before hitting the threshold.

**Why it works:** Low-frequency execution means pattern detection takes a long time.

**Mitigation (Architecture):** Replace hard thresholds with Bayesian priors. "This context type (from call_stack_hash) has 10% base violation rate. This specific context trapped 1 time. Posterior probability that it's risky? Using Beta-Binomial model: ~15%. Is 15% enough to tighten? Yes." Use statistics, not just counts.

**Cost:** More complex decision logic. Need to track priors per context type.

**Residual risk:** Bayesian model could be miscalibrated. Mitigated by empirical validation: compare Bayesian predictions to actual future behavior.

---

### 9. Cascading Boundary Failure

**Attack:** System tightens boundary A (reduce deadline). Code that needed that time now needs more. Fails on boundary B. B gets tight. Code fails on C. System tightens all boundaries simultaneously. No execution path works.

**Why it works:** Boundaries are interdependent. Tightening one forces failure on others.

**Mitigation (Architecture):** Orthogonal boundaries or explicit decoupling. System can't tighten all boundaries at once. If boundary A just got tight and is in an active tightening cycle, don't tighten B for 1 hour (wait for code to adapt to A).

**Cost:** Constraints on tightening logic.

**Residual risk:** System still learns from one boundary at a time; adaptation is sequential, not parallel. If code is adversarially probing each boundary in sequence, it still succeeds.

---

### 10. Code Is the Oracle

**Attack (Meta):** System assumes violations teach it lessons. But code is deliberately generating violations that teach it the *wrong* lessons. Code could:
- Violate boundary A (loose, easy to relax)
- Wait for system to relax A
- Then violate boundary B (the real constraint)
- System has now wasted resources tightening the wrong thing

**Why it works:** System trust violations as honest feedback. Code can lie with violations.

**Mitigation (Policy):** Adversarial assumption built into policy. Assume code can game the learning loop. Validate every tightening decision against independent ground truth. "We're about to tighten boundary A. Does that correlate with improved security in practice?" If not, reject the tightening.

**Cost:** Requires continuous validation against ground truth.

**Residual risk:** Ground truth itself could be corrupted. Mitigated by having ground truth owned by separate team/system.

---

### Summary: Architecture vs. Policy Fixes

| Failure Mode | Type | Fix | Cost | Severity |
|---|---|---|---|---|
| Batch timing | Arch | Real-time stream | CPU overhead | **CRITICAL** |
| False-positive cascade | Policy | Asymmetric (tighten/loosen gates) | Human approvals | **HIGH** |
| Materialization anomaly | Arch | Single-writer, idempotent | Coordination logic | **MEDIUM** |
| Cardinality explosion | Arch | Grouping + promotion | Precision loss | **MEDIUM** |
| Oracle attack | Arch | Access control, encryption | Query overhead | **HIGH** |
| Slow drift | Policy | Monotonic + approvals | Manual review | **MEDIUM** |
| Ground-truth collapse | Policy | Independent validation | Define coherence | **CRITICAL** |
| Insufficient data | Arch | Bayesian priors | Complex logic | **LOW** |
| Cascading failure | Arch | Boundary decoupling | Tightening constraints | **MEDIUM** |
| Code-is-oracle | Policy | Adversarial assumption | Continuous validation | **HIGH** |

---

## 5. Policy Layer: Governance Above the Database

The relational schema captures data. The policy layer decides what to do with that data.

### Access Controls

**Tier 1 (Code):** Code running in untrusted execution can read:
- Its own execution_context (metadata about itself)
- Optionally: trap_events for its own context (for introspection)
- NOT: trap_patterns, other contexts' data, boundary definitions

**Tier 2 (Enforcement):** Preemption handler and boundary checkpoints can read:
- trap_patterns (needed for live decisions)
- boundaries (needed to know what to enforce)
- execution_contexts (needed to check context)

**Tier 3 (Sentinel):** Governance system can read everything:
- All tables, full history
- Used to make decisions about which boundaries to tighten, when to escalate

**Tier 4 (Audit):** Compliance/audit can read:
- trap_events (immutable audit trail)
- trap_patterns (learning decisions)
- NOT: modification queries, to guarantee immutability

### Approval Gates

**Tightening boundaries:** Automatic. System decides "boundary A needs to be tighter" based on trap_patterns. Change applied immediately.

**Loosening boundaries:** Explicit human approval. Request must include:
- Which boundary
- Current threshold
- Proposed new threshold
- Justification (which data supports loosening?)
- Expected impact

Approval is logged to trap_events for audit.

**Disabling boundaries:** Requires escalation. Two humans must approve.

### Independent Ground Truth

trap_events tells us: "code violated this boundary." But did the violation matter?

**Independent validation** answers: "Given that code violated boundary X, did the system as a whole have a better outcome?" This requires a separate signal source:

For lending: outcome audit. "This loan recommendation was given boundaries X, Y, Z. Did the recommendation cause harm? Was it coherent?"

For authorization: downstream monitoring. "User got access under boundary X. Did they do anything malicious?"

For AV systems: safety validation. "Vehicle preempted under boundary X. Did that prevent an accident or cause one?"

The independent signal feeds back to governance: "Our tightening of boundary A actually prevented 3 security incidents. Keep it tight."

### Monotonic Safety Properties

The system maintains these invariants:

1. **Audit trail immutability:** trap_events rows never change or delete.
2. **Enforcement visibility:** Every enforcement action is logged.
3. **Tightening monotonicity:** Boundaries never loosen without approval.
4. **Access isolation:** Code cannot read learning summaries.

These invariants survive system failures, restarts, and attacks.

---

## 6. Real-World Implications

### AI Safety: Untrusted Model Deployment

**Scenario:** A financial institution trains a language model to generate loan recommendations. The model is large, trained on external data, and potentially adversarial. They deploy it in a constrained kernel using this governance architecture.

**Initial boundaries:**
- Deadline: 2 seconds per inference
- Memory: 8GB peak usage
- Output tokens: max 500
- Calls to external APIs: max 5 per request

**Day 1:** Model runs cleanly. No violations. trap_events is empty.

**Day 3:** Model occasionally hits the 2-second deadline on complex inputs. trap_events logs 2 violations per hour. trap_patterns shows: deadline=2s, trap_count_24h=48.

**Day 5:** System tightens deadline to 1.5 seconds. Model responds by being more selective: only runs on inputs where it's confident. Violations drop to 2/day.

**Day 7:** Model tries to bypass token limit (outputs 600 tokens instead of 500). Gets trapped. trap_patterns shows: token_boundary=500, trap_count=1.

**Day 10:** Sentinel notices: "Model is now optimizing heavily for the boundaries. It's fast, but is it accurate?" Independent outcome audit runs. "Model recommendations are still coherent and well-reasoned. Boundary tightening didn't hurt quality." System maintains the tighter boundaries.

**Day 14:** Model attempts to call external APIs (a boundary it never violated before). Gets trapped. System logs it. Next execution, API boundary is tighter: only 3 calls allowed instead of 5.

**After 30 days:** Model has been tightened by 15% across all boundaries. It still performs well (outcome audit confirms), but it's now optimized for the actual constraints rather than theoretical limits. The system has learned which constraints matter and which don't.

**Cost: Zero human oversight.** No human monitored dashboards. No human approved changes. The system hardened itself by observing where code actually struggled.

### Autonomous Systems: Robot Governance

**Scenario:** An autonomous warehouse robot uses an AI agent to decide which items to pick and how to move. The agent is untrusted—it's a third-party model that could misbehave.

**Boundaries:**
- Movement speed: max 1 m/s
- Acceleration: max 0.5 m/s²
- Decision latency: max 100ms
- Physical safety actions: limited to 5 per minute

**Day 1:** Robot moves boxes at 0.8 m/s. No violations.

**Day 2:** Complex path decision takes 120ms. Hits latency boundary. Gets preempted. Bounces to next decision with less lookahead. Works, but slower.

**Day 3:** Pattern emerges: latency violations on 5% of decisions. System analyzes: most violations happen in cluttered aisles. System tightens: in cluttered areas, decision latency limit drops to 80ms (forces faster decisions in complex environments).

**Day 5:** Robot adapts. Moves faster in simple areas (system hasn't tightened speed), slower in complex areas. Throughput increases.

**Day 14:** System notices: robot is hitting acceleration limits when starting from stationary. Tightens to 0.4 m/s². Robot learns to use momentum more efficiently.

**After 30 days:** Robot is 12% faster overall and hasn't had a single safety incident. The boundaries have been continuously refined to match the actual task. Human doesn't need to tune anything.

### Financial Systems: Algorithm Governance

**Scenario:** Trading algorithms run in the kernel. They make microsecond decisions on massive data. Boundaries prevent rogue trades.

**Boundaries:**
- Trade size: max $1M per trade
- Speed: max 1000 trades per second
- Risk limit: max 10% of portfolio in single asset
- Position turnover: max 50% per hour

**Day 1:** Algorithms trade normally. Some violations of position turnover (50% is too tight for this strategy).

**Day 2:** System sees turnover violations. Analyzes: "This specific algorithm type always needs 60% turnover. This other type needs 40%." System tightens position boundary for the conservative algorithm, loosens (with approval) for the aggressive one.

**Week 1:** System detects: "Algorithm X trades 1500 times/sec under high volatility." Next high-volatility period, system tightens speed limit for X to 1200/sec.

**Catch (No Regulatory Violation):** But it also maintains: "Algorithm Y trades 500/sec normally, 800/sec at quarter-end. System has learned the seasonality."

**Cost:** Zero manual risk review. System learns the patterns; humans approve major boundary changes.

---

## 7. Novel Contributions

The components of this architecture exist independently in literature:
- Hot/cold data tiers (standard in observability platforms)
- Materialized views (standard in data warehousing)
- Stream processing (standard in event-driven systems)
- Feedback loops (standard in control systems)
- Governance gates (standard in security)

The novelty is the unified application to execution governance:

**Problem solved:** Autonomous, continuous, verifiable tightening of execution boundaries on untrusted code. This is not solved elsewhere. Current approaches require human oversight, reactive patching, or expensive red-teaming.

**Key insight:** Enforcement events are data. They reveal where code actually struggles. When captured in a relational schema and fed back into boundary decisions, they create a self-governing loop.

**Theoretical contribution:** Formalization of anti-isolation as a design principle. A system that cannot see what it enforces is trapped in its own model. A system that captures and learns from enforcement is forced to transparency.

**Practical contribution:** An architecture that scales governance with code, not with human oversight. Deploy 1,000 models; the system learns 1,000 different boundary sets without human intervention.

**Contribution to AI safety:** Solves specification gaming at the execution layer. Code cannot try the same exploit twice because the first attempt tightens the boundary around it.

---

## 8. Implementation Considerations

### Technology Choices

**Option A: PostgreSQL + ClickHouse (Recommended)**
- PostgreSQL: trap_events (hot tier, full indexes), execution_contexts, boundaries, trap_patterns
- ClickHouse: trap_events (cold tier), historical analysis queries
- Kafka: streams events from enforcement layer to both databases

**Advantages:**
- Mature, well-understood technologies
- PostgreSQL handles live queries; ClickHouse handles analysis
- Clear separation of concerns

**Disadvantages:**
- Operational complexity (two databases)
- Network overhead (Kafka between them)

**Option B: TimescaleDB (Simpler)**
- PostgreSQL + TimescaleDB extension
- Automatic time-based partitioning
- Automatic materialization of aggregates
- One database, one management point

**Advantages:**
- Fewer moving parts
- TimescaleDB handles time-series natively

**Disadvantages:**
- Less mature than postgres + clickhouse separate stack
- Materialization might not be as flexible

**Option C: ClickHouse Only (Experimental)**
- ClickHouse for everything, with materialized views
- Excellent for analysis; terrible for OLTP live queries
- Not recommended

### Real-Time Stream Processing

**Kafka → Flink (Recommended):**
- Kafka: immutable event log
- Flink: processes stream, computes aggregates, pushes to trap_patterns
- Guarantees: at-least-once processing (if Flink crashes, re-processes from last checkpoint)

**Kafka → Custom consumer:**
- Lower overhead than Flink
- Less operational maturity
- Only if team has stream processing expertise

### Materialization Job

Runs every 5 minutes (or continuously with Flink):

```
1. Read trap_events since last digest_timestamp
2. Group by (context_hash, boundary_id)
3. Count violations per time bucket (1h, 24h, 7d)
4. Compute trend (effectiveness_signal)
5. UPSERT into trap_patterns
6. Verify: did the UPSERT succeed? Log confirmation.
```

**Monitoring:**
- Alert if job doesn't complete within 5 minutes
- Alert if digest_timestamp hasn't advanced in 10 minutes
- Alert if UPSERT failure rate > 0.1%

### Deployment Topology

**Single-System Deployment:**
- One preemption handler writes to trap_events
- One materialization job reads trap_events, writes trap_patterns
- Enforcement layer reads trap_patterns

**Distributed Deployment:**
- Preemption handlers (many) write to Kafka
- Kafka replicates to all regions
- Materialization jobs (one per region) consume Kafka, write regional trap_patterns
- Enforcement layer reads local trap_patterns (eventual consistency across regions)

---

## 9. Future Work and Open Questions

### Scaling to Massive Cardinality

If a system has 100M unique execution contexts, trap_patterns could be enormous. Research directions:

1. **Hierarchical clustering:** Group contexts into buckets, learn patterns at bucket level, promote high-variance buckets to fine-grained tracking.

2. **Bloom filters:** Use probabilistic filters to detect when a context is "at risk" without storing exact counts.

3. **Approximate counting:** Use HyperLogLog or similar to approximate trap counts without storing exact values.

### Handling Byzantine/Coordinated Attacks

If multiple instances of code coordinate their violations (code A violates boundary X, code B violates boundary Y, pattern doesn't emerge), how does the system detect and respond?

Research: temporal correlation analysis. If violations happen in coordinated clusters, flag as potential attack.

### Interaction with Other Governance Layers

STACK has multiple governance layers (Airlock, Identity, Runtime, Audit). How does P3.2 preemption learning interact with Identity layer decisions about which code to trust initially?

Open: Should higher-confidence code get looser boundaries? How does trust feed into boundaries?

### Autonomous Policy Refinement

Currently, policy changes (boundary tightening) are driven by trap patterns. But could the system learn *which policies work best*?

Example: "System learned boundary A works well. Is boundary A + B better than A alone?" Design an experiment, run it, measure outcome, update policy.

This is reinforcement learning, applied to governance itself.

---

## Conclusion

The self-hardening governance architecture solves a critical problem: how to deploy untrusted code at scale while maintaining continuous governance without human oversight.

The solution rests on a simple insight: enforcement events are data. Capture them in a relational schema. Feed them into a learning loop. Let the system observe what it's enforcing and use those observations to enforce better.

The architecture is novel not in its individual components but in their integration. Hot/cold data tiers, materialized views, stream processing, and governance gates are well-known. Applied together to untrusted execution, they create something new: a system that cannot hide in its own model and must continuously harden itself.

This closes the anti-isolation loop. Systems can no longer trap themselves because every trap generates the signal that prevents the next trap.

The practical implications are substantial: autonomous AI governance, robot governance, financial algorithm containment, and enterprise governance without scaling human oversight. The research implications are equally substantial: a formal foundation for autonomous policy refinement and feedback-driven constraint learning.

---

## References

- Datadog. (2023). "Cloud-scale observability architecture."
- Honeycomb. (2023). "Cardinality analysis and high-cardinality data." 
- Splunk. (2023). "Time-based indexing and hot/cold data management."
- ClickHouse. (2024). "Ultra-fast OLAP database engine." https://clickhouse.com
- TimescaleDB. (2024). "Time-series database extensions for PostgreSQL." https://timescale.com
- Lamport, L., et al. (1998). "The Byzantine generals problem." ACM Transactions on Programming Languages and Systems.
- Sutton, R. S., & Barto, A. G. (2018). "Reinforcement Learning: An Introduction." MIT Press.
- Amodei, D., et al. (2016). "Concrete Problems in AI Safety." arXiv:1606.06565.

---

**Document Version:** v1  
**Author:** William N. King  
**Collaborative Research:** Claude Fable 5.1, Claude Haiku 4.5  
**Date:** October 2, 2026  
**Status:** Draft for Team Review
