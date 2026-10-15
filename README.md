# P3.2 Enforcement Telemetry & Asymmetric Adaptive Authority

**Status:** Experimental Research Proposal  
**Author:** William N. King  
**Collaborators:** Claude Fable 5.1, Claude Haiku 4.5  
**Last Updated:** October 14, 2026

---

## Overview

This repository contains research on a governance architecture for untrusted code execution based on three core ideas:

1. **Enforcement as telemetry:** Boundary violations are treated as observations about the relationship between workloads and constraints, not merely as failures to contain.

2. **Asymmetric adaptive authority:** Observed enforcement pressure permits automatic boundary tightening, but absence of pressure does NOT permit autonomous loosening. This creates a one-way safety ratchet.

3. **Interpretation layer:** Between observing violations and deciding to tighten boundaries, there must be a layer that asks "what does this violation actually mean?" and validates that proposed tightening improves security.

---

## What This Is NOT

- Not an established architecture
- Not a proven mechanism
- Not production-ready
- Not a solution to execution governance (not yet)

---

## What This Is

An architectural proposal structured around three research questions:

### A. Detection
Can we reliably identify meaningful patterns in enforcement events without generating false positives or missing real signals?

### B. Adaptation  
Can we safely transform detected patterns into constraint changes without allowing adversarial code to game the learning loop?

### C. Validation
Can we establish that constraint changes actually improve security (rather than just reducing visible violations)?

---

## Repository Structure

```
p3-2-governance-research/
├── README.md                              (this file)
├── RESEARCH_QUESTIONS.md                  (detailed breakdown of A/B/C)
├── CRITIQUE.md                            (critical assessment of the proposal)
├── ARCHITECTURE.md                        (architectural overview)
├── /docs/
│   ├── white-paper-v1.md                  (full technical proposal)
│   ├── relational-schema.md               (database design with caveats)
│   ├── interpretation-layer-design.md     (the missing piece)
│   └── asymmetric-authority-safety.md     (why tighten-only matters)
├── /experiments/
│   ├── adversarial-testbed.md             (how to validate against gaming)
│   └── validation-framework.md            (how to prove tightening works)
└── /implementation/
    ├── technology-choices.md              (PostgreSQL, TimescaleDB, ClickHouse, Kafka, Flink)
    └── materialization-correctness.md     (fixes to pseudocode issues)
```

---

## Reading Order

**Start here:**
1. RESEARCH_QUESTIONS.md (clarifies what's actually being tested)
2. CRITIQUE.md (honest assessment of overclaiming and gaps)

**Then:**
3. ARCHITECTURE.md (what the system actually does)
4. docs/asymmetric-authority-safety.md (the strongest piece)
5. docs/interpretation-layer-design.md (what's missing)

**If diving deeper:**
6. docs/white-paper-v1.md (full proposal)
7. experiments/adversarial-testbed.md (how to invalidate it)

---

## Core Insight

The interesting contribution is NOT a database architecture.

It is: **Can a governance system maintain safety properties through asymmetric authority (autonomous tightening, human-gated loosening) combined with an interpretation layer that prevents violations from being misleading feedback?**

If that works, governance scales from "human watches every execution" to "human adjudicates constraint accumulation quarterly."

If it doesn't, the research clarifies where and why.

---

## Evidence so far (October 14, 2026)

The three questions have been worked in the `experimental` repository (github.com/wking53214/experimental; PR #5 until merged), with pre-registered criteria and predictions written before each run. Where things stand:

- **A. Detection.** Statistical detection on real telemetry cries wolf: false alarms spend the tightening budget before real trouble arrives (criteria 3 and 4 fail). Enforcement events as the signal work for runaways and bounded attackers, but no trap-count rule separates a legitimately wide-tailed agent from a mild runaway. See `docs/interpretation-layer-design.md`.
- **B. Adaptation.** The one-way ratchet holds under fuzzing, soak runs, twelve rounds of trace replays and a six-reviewer red team (two real enforcement fixes came out of it): automation never loosened a limit, every change is attributable, limits survive restarts and outside signals. The tightening budget caps both the damage an attacker can do through it (about 27% of a limit) and the benefit it buys (the same 27%). Taken apart against a two-way limiter, the cap and the interpretation rules do all the measured protecting; the one-way direction's measured benefit was a cold-start artefact and that claim is withdrawn. See `docs/asymmetric-authority-safety.md`, `experiments/adversarial-testbed.md`, `experimental/docs/RED_TEAM.md`.
- **C. Validation.** Built as a canary A/B comparison in time currency with confidence bounds. It confirms gross harm within a few slices, marginal harm late, and benefit almost never at a 10% canary; it is blind to intent. Criterion 10 fails on confirmation speed and passes everything else. See `experiments/validation-framework.md`.

**The claim, as revised.** The strong claim, that the system judges on its own when a limit should come down, was tested and did not hold. The project now claims bounded autonomy with human-gated loosening and validated tightening: human effort relocated to adjudication, not removed. Nothing has run on real traffic.

**What comes next.** `docs/program-priced-accountable-limiting.md` (2026-10-15) re-aims the thesis at what the data supports: a limiter that prices every tightening in time, owes an outcome on each one, reports which program classes it harms, and is scored on a public benchmark built from the cases that broke the first design. It names the capabilities reused from this portfolio (outcome obligations, the disparate-impact screen, the SWIZZLE and ASSAY harness mechanisms, URE's recovery rules), the open problem at its center (tightening censors its own evidence), and the pre-registered predictions each phase must meet. Its section 8 is a documented novelty check against the re-aimed thesis: each pillar has adjacent prior art, three have close 2026 precedents, and no found work combines them or measures the self-censoring loop.

## Known Issues

1. **Overclaiming:** Initial white paper claims the system "solves" execution governance. It doesn't. The experimental work has since tested the strong claim and it did not hold; see "Evidence so far".

2. **Interpretation layer:** built as a set of rules (expected load, rate, drift, excess release) that answer the cheapest version of "what does this violation mean?" and are documented with what they cannot separate (`docs/interpretation-layer-design.md`).

3. **Validation:** a canary-based mechanism now exists and is measured; it confirms harm, not benefit, at the tested canary volume (`experiments/validation-framework.md`).

4. **Constraint accumulation:** Asymmetric authority prevents autonomous loosening, which means constraints accumulate forever unless humans periodically adjudicate them. This is a feature (safety), but it's not "without human oversight."

5. **Technical debt:** 
   - Materialization checkpoint logic (MAX timestamp) is unsafe; fix in `implementation/materialization-correctness.md`
   - "effectiveness_score" now defined for deadline boundaries only, as a bounded verdict rather than a float (`docs/relational-schema.md`)
   - Pseudocode has bugs

---

## Next Steps

The architecture proposal survives only if it can answer the three research questions:

1. **Build a Detection subsystem** that identifies patterns without false positives
2. **Design an Adaptation subsystem** that proposes boundary changes from those patterns
3. **Implement a Validation subsystem** that proves tightening improves security

If all three work together, even against adversarial workloads deliberately trying to game the learning loop, then the architecture has merit.

If one fails, the research clarifies why and points to a different approach.

---

## Contact

Questions, feedback, or critiques: see CRITIQUE.md as the model for rigorous dissection.

---

**Document version:** Experimental v1  
**Repo status:** Research proposal, not production  
**Approval required:** Before any deployment beyond research
