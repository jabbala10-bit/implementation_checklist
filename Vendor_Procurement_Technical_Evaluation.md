# Vendor & Procurement Technical Evaluation — RFP Scoring, Build-vs-Buy Frameworks & POC Methodology

**Principal Engineer / AI Architect / FDE Reference · Gunasekar Jabbala**

> The AI Governance document's Section 10.1-10.2 touched build-vs-buy and vendor selection briefly. This document is the formalized version of that skill — the actual evaluation methodology an FDE or Principal Architect uses when a client or organization needs to choose between vendors, between building internally, and between competing technical approaches, with enough rigor that the decision survives scrutiny months later when someone asks why this choice was made.

---

## Table of Contents

1. [The Vendor Evaluation Maturity Model](#1-the-vendor-evaluation-maturity-model)
2. [Formalized Build-vs-Buy Analysis](#2-formalized-build-vs-buy-analysis)
3. [RFP Technical Scoring Methodology](#3-rfp-technical-scoring-methodology)
4. [Proof-of-Concept (POC) Evaluation Design](#4-proof-of-concept-poc-evaluation-design)
5. [Vendor Risk Assessment Beyond Technical Fit](#5-vendor-risk-assessment-beyond-technical-fit)
6. [The FDE-Specific Skill: Running This Process With a Client, Not Just for One](#6-the-fde-specific-skill-running-this-process-with-a-client-not-just-for-one)
7. [Complexity Reduction for Vendor Evaluation Specifically](#7-complexity-reduction-for-vendor-evaluation-specifically)
8. [Decision Framework](#8-decision-framework)

---

## 1. The Vendor Evaluation Maturity Model

| Level | Name | Capabilities |
|---|---|---|
| **1** | Opinion-Based Selection | Vendor chosen based on familiarity, sales relationship, or general reputation, with no structured comparison |
| **2** | Checklist Comparison | A feature checklist comparison across vendors, but no weighting, no empirical validation |
| **3** | Weighted, Evidence-Based Evaluation | Explicit weighted scoring criteria, empirical POC validation against actual representative workloads, documented decision rationale |
| **4** | Portfolio-Level Vendor Strategy | Vendor decisions made in the context of a broader portfolio strategy, avoiding excessive concentration risk, with planned exit strategies and continuous re-evaluation as the vendor landscape evolves |

This is a recurring weak spot industry-wide — even technically sophisticated organizations frequently make Level 1-2 vendor decisions for genuinely consequential choices, because the rigor that goes into internal architecture decisions often isn't applied with the same discipline to external vendor decisions, despite the stakes often being comparable or higher.

---

## 2. Formalized Build-vs-Buy Analysis

### 2.1 The Total Cost of Ownership (TCO) Framework, Done Properly

**Principal-level note:** the most common build-vs-buy mistake is comparing a vendor's sticker price against only the visible build cost, initial engineering time, ignoring the much larger set of ongoing costs on both sides. A proper TCO comparison includes, for both options: initial implementation cost, ongoing maintenance and support cost, opportunity cost of engineering time spent (what else those engineers could have built), and the cost of eventual migration or exit if the choice doesn't work out.

```json
{
  "build_vs_buy_tco": {
    "decision": "EU AI Act compliance documentation tooling",
    "build_option": {
      "initial_engineering_cost_weeks": 8,
      "ongoing_maintenance_weeks_per_year": 4,
      "opportunity_cost": "8 weeks of senior engineering time not spent on the core product roadmap",
      "exit_cost": "low, since it's owned IP with no vendor dependency"
    },
    "buy_option": {
      "initial_integration_cost_weeks": 1,
      "annual_license_cost_usd": 45000,
      "ongoing_maintenance_weeks_per_year": 0.5,
      "exit_cost": "moderate, would require migrating documented data out of the vendor's format"
    },
    "decision": "buy, given the engineering opportunity cost and the fact that compliance tooling isn't a core differentiator worth building in-house"
  }
}
```

**Principal-level note:** this is the formal, structured version of the AI Governance document's brief build-vs-buy mention — note specifically that opportunity_cost is being treated as a real, comparable cost figure rather than an intangible afterthought, which connects directly to the FinOps document's principle of converting operational and time costs into comparable financial terms rather than only comparing the costs that happen to already be expressed in dollars.

### 2.2 The Core vs. Context Heuristic for Resolving Ambiguous Cases

**Principal-level note:** a useful heuristic when TCO numbers are close: is this capability part of your organization's actual competitive differentiation, core, or is it necessary infrastructure that every competitor also needs and that doesn't differentiate you, context? Building is more justified for core capabilities where owning the IP and having full control matters strategically; buying is more justified for context capabilities where a mature vendor has already solved the problem better than a first internal attempt likely would, and your engineering effort is better spent on what actually differentiates you.

---

## 3. RFP Technical Scoring Methodology

### 3.1 Weighted Criteria - Making the Scoring Defensible, Not Just a Gut Feeling Dressed Up as a Number

```json
{
  "rfp_scoring_framework": {
    "evaluation": "vector database vendor selection",
    "criteria": [
      { "criterion": "query latency at target scale", "weight": 0.25, "measurement": "empirical POC benchmark" },
      { "criterion": "data residency and compliance fit", "weight": 0.20, "measurement": "documented certification review against governance requirements" },
      { "criterion": "total cost of ownership at projected scale", "weight": 0.20, "measurement": "TCO model" },
      { "criterion": "operational maturity, SLA and support responsiveness", "weight": 0.15, "measurement": "reference customer interviews" },
      { "criterion": "ecosystem and integration fit with existing stack", "weight": 0.20, "measurement": "technical integration spike" }
    ],
    "scores_per_vendor": { "vendor_a": 8.4, "vendor_b": 7.1, "vendor_c": 6.8 }
  }
}
```

**Principal-level note:** the weights themselves are the most important and most frequently under-discussed part of this framework — they should be agreed upon by stakeholders before vendor scores are known, not adjusted afterward to favor whichever vendor stakeholders already preferred informally. Setting weights after seeing preliminary scores is a subtle but serious form of motivated reasoning that undermines the entire point of having a structured framework; this is worth naming explicitly as a process discipline, since it's a very human failure mode that even well-intentioned evaluation teams fall into without noticing.

### 3.2 Weighting Criteria That Resist Easy Quantification

**Principal-level note:** ecosystem and integration fit and operational maturity are harder to score objectively than raw latency benchmarks, but excluding them from the weighted framework just because they're harder to quantify produces a framework that systematically overweights what's easy to measure rather than what actually matters — the correct response is finding a workable proxy measurement, a technical integration spike, structured reference customer interviews with a consistent question set, rather than dropping the criterion or leaving it as an unweighted gut-check appended after the real scoring.

---

## 4. Proof-of-Concept (POC) Evaluation Design

### 4.1 The Single Most Important POC Design Principle - Representative Data and Load

**Principal-level note:** the most common POC evaluation failure is testing against clean, small, vendor-provided sample data rather than your own actual representative data and load — a vector database that performs beautifully on a vendor's curated demo dataset can behave very differently against your actual large corpus with your actual query distribution's specific characteristics, the same emphasis on evaluation using real production-distribution data found in the RAG document applies directly to vendor POC evaluation as well.

```json
{
  "poc_design": {
    "vendor_evaluated": "vendor_a_vector_database",
    "test_data": "anonymized_sample_of_actual_production_corpus, not vendor demo data",
    "test_query_distribution": "sampled from actual historical production query logs, not synthetic queries",
    "test_scale": "10% of projected production scale, extrapolated with documented assumptions for full scale",
    "success_criteria_defined_before_test": ["p95 latency under 200ms at test scale", "recall above 0.92 against the labeled evaluation set"]
  }
}
```

**Principal-level note:** success_criteria_defined_before_test is the critical discipline — defining what passing looks like before running the POC prevents the same motivated-reasoning risk flagged in Section 3.1's weighting discussion, where success criteria might otherwise get quietly adjusted after seeing results to justify whichever vendor was already favored going into the evaluation.

### 4.2 Running Multiple Vendors' POCs Under Identical Conditions

**Principal-level note:** a POC comparison is only valid if every vendor is evaluated against the same test data, the same query distribution, and the same success criteria — running vendor A's POC against one dataset and vendor B's POC against a slightly different, improved dataset introduces a confound that makes the comparison meaningless, even if each individual POC was well-designed in isolation.

---

## 5. Vendor Risk Assessment Beyond Technical Fit

### 5.1 Vendor Stability and Continuity Risk

**Principal-level note:** this connects directly to the Model Serving document's vendor risk assessment for AI providers specifically and the Security Architecture document's supply chain concerns — a technically excellent vendor that's a single-founder startup with uncertain funding runway represents a continuity risk that a pure technical scoring framework wouldn't surface at all, and needs its own explicit assessment dimension.

```json
{
  "vendor_continuity_risk": {
    "vendor": "vendor_a",
    "funding_stage": "series_b",
    "years_in_market": 4,
    "customer_concentration_risk": "low, broad enterprise customer base disclosed in public materials",
    "exit_strategy_if_vendor_fails": "documented migration plan to vendor_b as a fallback, estimated 6-week migration effort",
    "contract_terms_include_data_portability_guarantee": true
  }
}
```

**Principal-level note:** exit_strategy_if_vendor_fails being a pre-documented plan, not a "we'd figure it out" assumption, is what separates genuine risk management from optimistic hand-waving — having this answer ready before signing a vendor contract is a specific, concrete deliverable worth producing as part of any consequential vendor evaluation, not just a mental note.

### 5.2 Negotiating Contractual Terms That Protect Against the Risks Identified

**Principal-level note:** the vendor risk assessment should directly inform contract negotiation — data portability guarantees, defined SLA terms with real remedies rather than just aspirational targets, and notification requirements for material changes such as acquisition, pricing changes, or deprecation of features you depend on, are all things to negotiate into the contract based on the specific risks Section 5.1 surfaced, rather than discovering their absence only when a risk materializes.

---

## 6. The FDE-Specific Skill: Running This Process With a Client, Not Just for One

### 6.1 The Added Complexity of Facilitating, Not Just Deciding

**Principal-level note:** when running this evaluation process for a client rather than for your own organization, the FDE-specific skill is facilitating a structured, defensible process while the client's own stakeholders hold the actual decision authority — your job is ensuring the weighting, the POC design, and the risk assessment are rigorous and unbiased, even when you may have your own professional opinion about which vendor is genuinely best, since an FDE's credibility depends on the client trusting the process was fair, not just trusting that you personally picked well.

### 6.2 Documenting the Process for the Client's Own Future Reference

**Principal-level note:** this connects directly to the FDE Coding Challenges document's recurring theme of leaving the client's own team better equipped than before the engagement — a well-documented vendor evaluation framework, with the weighting rationale and POC methodology explicitly recorded, is something the client's team can reuse for their next vendor decision without external help, which is a meaningfully higher-value deliverable than just the recommendation itself.

---

## 7. Complexity Reduction for Vendor Evaluation Specifically

| Degree of Freedom | Reduction Strategy |
|---|---|
| Number of evaluation criteria | A focused set of 5-7 weighted criteria genuinely driving the decision, not an exhaustive checklist where most items don't actually differentiate vendors |
| POC scope | A representative, bounded POC scenario validating the highest-risk assumptions, not an attempt to fully replicate production in miniature |
| Vendor shortlist size | Narrow to 2-3 serious finalists before investing in deep POC effort, not running expensive full evaluations against every vendor that technically qualifies |
| Risk assessment depth | Proportional to the decision's actual consequence and reversibility — deep risk assessment for hard-to-reverse, high-dependency choices; lighter assessment for easily-reversible, low-stakes ones |

---

## 8. Decision Framework

1. Has build-vs-buy been analyzed with a genuine TCO comparison including opportunity cost and exit cost, or only by comparing a vendor's sticker price against the visible portion of build cost?
2. Were the RFP scoring weights agreed upon before vendor scores were known, or adjusted afterward to favor an already-preferred vendor?
3. Was the POC run against your own representative data and query distribution, or against vendor-provided demo data that may not reflect real production behavior?
4. Has vendor continuity and concentration risk been assessed explicitly, with a documented exit strategy, or only technical fit?
5. If running this process for a client as an FDE, is the process itself documented well enough that the client's team could repeat it independently for their next vendor decision?

**The governing test:** a vendor or build-vs-buy decision should be as defensible and reconstructable months later as any internal architecture decision recorded in an ADR, with documented weighted criteria, empirical evidence from representative testing, and an explicit risk assessment, rather than relying on the decision-makers' memory of why it felt right at the time, which is precisely the same auditability standard this entire document series applies to every other category of consequential technical decision.

---

## Companion Documents

Part of the Principal Engineer / AI Architect / FDE core engineering series. This document closes the loop on a thread present since the AI Governance document:

- `AI_Governance_Compliance_Schemas.md` — the brief build-vs-buy and vendor selection sections this document formalizes into a complete methodology
- `Model_Serving_Architecture_Deep_Dive.md` — the AI provider vendor risk assessment this document's continuity risk framework extends generally
- `Security_Architecture_Beyond_Agent_IAM.md` — the supply chain risk concerns directly relevant to vendor continuity assessment
- `FinOps_Cost_Engineering.md` — the opportunity-cost-as-comparable-currency principle this document's TCO framework directly applies
- `RAG_Architecture_Deep_Dive.md` — the representative-evaluation-data principle this document's POC design section directly extends to vendor testing
- `Principal_AI_FDE_Coding_Challenges.md` — the FDE client-engagement value-add theme this document's Section 6 directly continues
