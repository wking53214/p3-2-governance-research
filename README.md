# P3.2 Enforcement Telemetry & Asymmetric Adaptive Authority

**Status:** Experimental Research Proposal  
**Author:** William N. King  
**Collaborators:** Claude Fable 5.1, Claude Haiku 4.5  
**Last Updated:** October 2, 2026

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
├── ARCHITECTURE.md                        (architectural overview and known gaps)
├── COMMENTARY.md                          (synthesis commentary)
├── PUSH_TO_GITHUB.md                      (setup notes for publishing this repo)
└── docs/
    └── white-paper-v1.md                  (full technical proposal; sec. 3 flywheel, sec. 4 red-team, sec. 5 policy layer)
```

The prototype that tests the asymmetric-authority rule lives in a separate repository,
`wking53214/experimental`. Its `docs/THEORY_ALIGNMENT.md` maps each claim here to what the
code implements, partially implements, or leaves out.

---

## Reading Order

**Start here:**
1. RESEARCH_QUESTIONS.md (clarifies what's actually being tested)
2. CRITIQUE.md (honest assessment of overclaiming and gaps)

**Then:**
3. ARCHITECTURE.md (what the system actually does)
4. docs/white-paper-v1.md, sec. 5 "Policy Layer" (the asymmetric-authority rules and approval gates)
5. docs/white-paper-v1.md, sec. 4 "Red-Team Analysis" (failure modes and mitigations)

**If diving deeper:**
6. docs/white-paper-v1.md (full proposal)
7. The `experimental` repository's `docs/LIMITATIONS.md` and `docs/THEORY_ALIGNMENT.md` (what the prototype does and does not do)

---

## Core Insight

The interesting contribution is NOT a database architecture.

It is: **Can a governance system maintain safety properties through asymmetric authority (autonomous tightening, human-gated loosening) combined with an interpretation layer that prevents violations from being misleading feedback?**

If that works, governance scales from "human watches every execution" to "human adjudicates constraint accumulation quarterly."

If it doesn't, the research clarifies where and why.

---

## Known Issues

1. **Overclaiming:** Initial white paper claims the system "solves" execution governance. It doesn't. It's a proposal.

2. **Missing interpretation layer:** Current design jumps from "violation happened" → "tighten boundary." Needs an intermediate layer asking "what does this violation mean?"

3. **Unproven validation:** No mechanism established for validating that boundary tightening actually improves security (vs. just reducing violation counts).

4. **Constraint accumulation:** Asymmetric authority prevents autonomous loosening, which means constraints accumulate forever unless humans periodically adjudicate them. This is a feature (safety), but it's not "without human oversight."

5. **Technical debt:** 
   - Materialization checkpoint logic (MAX timestamp) is unsafe
   - "effectiveness_score" undefined
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
