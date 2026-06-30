# AI Governance & Compliance Schemas — EU AI Act, DORA, NIS2 Evidence Architecture

**Principal AI Engineer / FDE Reference · Gunasekar Jabbala**

> Compliance fails operationally, not legally, most of the time — not because the regulation was misunderstood, but because the evidence needed to demonstrate compliance was never captured in a structured, retrievable form. This document treats regulatory obligations the same way the rest of this series treats agent messages: as explicit, versioned, auditable contracts, generated as a byproduct of normal system operation rather than reconstructed reactively before an audit.

---

## Table of Contents

1. [The Compliance Architecture Maturity Model](#1-the-compliance-architecture-maturity-model)
2. [Risk Classification Schemas](#2-risk-classification-schemas)
3. [Evidence & Audit Trail Contracts](#3-evidence--audit-trail-contracts)
4. [Multi-Framework Incident Mapping Schemas](#4-multi-framework-incident-mapping-schemas)
5. [Conformity & Documentation Lifecycle](#5-conformity--documentation-lifecycle)
6. [Complexity Reduction for Governance Specifically](#6-complexity-reduction-for-governance-specifically)
7. [Decision Framework](#7-decision-framework)

---

## 1. The Compliance Architecture Maturity Model

| Level | Name | Capabilities |
|---|---|---|
| **1** | Document-Based Compliance | Static policy documents, manual classification, compliance disconnected from engineering |
| **2** | Tracked Compliance | AI system inventory register, versioned classification, periodic manual review |
| **3** | Operationalized Compliance | Classification re-evaluated on system change, automated evidence capture, regulatory mapping built into incident response |
| **4** | Continuous Compliance Platform | Real-time conformity monitoring, automated audit-ready evidence generation, cross-framework evidence reuse, regulatory-change-aware policy updates |

The gap between Level 1 and Level 3-4 is exactly the gap between "we have a compliance document" and "we could pass a surprise audit tomorrow using only system-generated evidence." Most organizations significantly overestimate which level they're actually at.

---

## 2. Risk Classification Schemas

### 2.1 AI System Inventory Record

```json
{
  "ai_system_inventory_record": {
    "system_id": "sys_fraud_detection_v3",
    "system_name": "Real-Time Fraud Scoring Engine",
    "intended_purpose": "Score transactions for fraud risk to inform block/allow decisions",
    "deployment_context": "consumer_banking_transactions_eu",
    "data_categories_processed": ["financial_transaction_data", "behavioral_data"],
    "owner_role": "fraud_engineering_lead",
    "classification_status": "high_risk",
    "classification_version": "3",
    "last_reviewed_at": "2026-05-01T00:00:00Z",
    "next_review_due": "2026-11-01T00:00:00Z"
  }
}
```

- **Principal-level note:** This record is the foundation everything else in this document builds on — you cannot classify, document, or audit a system you haven't catalogued. `next_review_due` being a tracked field (not a manual calendar reminder somewhere) is what makes classification a living process rather than a one-time exercise.

### 2.2 EU AI Act Risk Classification Decision Record

```json
{
  "risk_classification": {
    "system_id": "sys_fraud_detection_v3",
    "framework": "eu_ai_act",
    "annex_iii_category_evaluated": "access_to_essential_services",
    "classification_result": "high_risk",
    "reasoning": "System decisions can result in account restriction or service denial, directly affecting consumer access to financial services",
    "classified_by": "compliance_architect_role",
    "classification_date": "2026-05-01T00:00:00Z",
    "reviewed_by_legal": true,
    "conservative_default_applied": false
  }
}
```

- **Principal-level note:** `conservative_default_applied` is worth including explicitly — when a classification call is genuinely borderline, documenting that you deliberately chose the more conservative (higher-risk) classification is itself part of a defensible compliance posture. An incorrect classification in the high-risk direction is far cheaper to have made than one in the other direction.

### 2.3 Provider vs. Deployer Obligation Mapping

```json
{
  "obligation_mapping": {
    "system_id": "sys_fraud_detection_v3",
    "organization_role": "provider_and_deployer",
    "provider_obligations": ["technical_documentation", "conformity_assessment", "ce_marking"],
    "deployer_obligations": ["fundamental_rights_impact_assessment", "human_oversight_operational", "post_market_monitoring"],
    "obligations_satisfied": ["technical_documentation", "human_oversight_operational"],
    "obligations_outstanding": ["fundamental_rights_impact_assessment"]
  }
}
```

- **Principal-level note:** Building an internal system makes you both provider and deployer simultaneously, which means *both* obligation sets apply in full — this is the most commonly missed nuance, and tracking `obligations_outstanding` explicitly as a structured field (not buried in a narrative compliance memo) is what makes the gap visible and actionable.

### 2.4 GPAI-Embedded-in-High-Risk-System Layering

```json
{
  "layered_classification": {
    "system_id": "sys_customer_assistant_v2",
    "embedded_gpai_provider": "external_foundation_model_vendor",
    "gpai_technical_documentation_received": true,
    "system_level_classification": "high_risk",
    "system_level_documentation_independent_of_gpai": true,
    "note": "GPAI provider documentation and deployer high-risk obligations tracked as separate, non-substitutable evidence layers"
  }
}
```

- **Principal-level note:** These two documentation layers don't merge into one assessment — a vendor's GPAI technical documentation doesn't discharge your own deployer-side high-risk obligations, and conflating them is a common and risky shortcut.
---

## 3. Evidence & Audit Trail Contracts

### 3.1 Human Oversight Operational Evidence

```json
{
  "human_oversight_record": {
    "system_id": "sys_fraud_detection_v3",
    "decision_id": "dec_88210",
    "ai_recommendation": "block_transaction",
    "human_reviewer": "analyst_user_2210",
    "human_action": "confirmed_block",
    "review_duration_seconds": 45,
    "reviewed_at": "2026-06-21T12:00:00Z"
  }
}
```

- **Principal-level note:** This is the single most commonly missing piece of documentation found in companies that believe they're compliant — human oversight is documented as a *policy statement* ("a human reviews high-risk decisions") but not as an *operational record*. Auditors want evidence the oversight actually happened on specific decisions, not just that it was promised in a policy document.

### 3.2 Fundamental Rights Impact Assessment (FRIA) Record

```json
{
  "fria_record": {
    "system_id": "sys_fraud_detection_v3",
    "affected_population": "consumer_banking_customers_eu",
    "fundamental_rights_at_risk": ["non_discrimination", "access_to_financial_services"],
    "mitigation_measures": ["bias_testing_quarterly", "human_review_for_high_value_blocks", "appeal_process_available"],
    "residual_risk_acceptance": {
      "accepted_by": "chief_risk_officer_role",
      "accepted_at": "2026-04-01T00:00:00Z",
      "rationale": "Mitigations reduce risk to acceptable level given essential fraud prevention need"
    }
  }
}
```

- **Principal-level note:** A FRIA without an explicit, named residual risk acceptance is incomplete — every mitigation reduces risk but rarely to zero, and someone with actual authority needs to own the decision that what remains is acceptable. A generic template without this specific finding won't hold up under scrutiny.

### 3.3 Bias & Fairness Testing Evidence

```json
{
  "bias_test_record": {
    "system_id": "sys_fraud_detection_v3",
    "test_date": "2026-05-15T00:00:00Z",
    "dimensions_tested": ["geographic_region", "account_tenure"],
    "disparate_impact_detected": false,
    "false_positive_rate_by_segment": { "segment_a": 0.021, "segment_b": 0.024 },
    "tested_by": "independent_data_science_reviewer"
  }
}
```

- **Principal-level note:** `tested_by: independent_data_science_reviewer` matters — bias testing conducted by the same team that built the system carries less evidentiary weight than independent review. Where possible, structure this as a genuinely separate review function, not a self-certification.

### 3.4 Immutable Audit Log Entry (Append-Only)

```json
{
  "audit_log_entry": {
    "entry_id": "audit_99021",
    "entry_hash": "sha256:a1b2c3...",
    "previous_entry_hash": "sha256:f9e8d7...",
    "event_type": "classification_changed",
    "system_id": "sys_fraud_detection_v3",
    "actor": "compliance_architect_role",
    "timestamp": "2026-05-01T00:00:00Z",
    "immutable": true
  }
}
```

- **Principal-level note:** The hash-chaining (`entry_hash` incorporating `previous_entry_hash`) is what makes tampering detectable rather than just procedurally discouraged — any modification to a past entry breaks the chain for every subsequent entry, which is verifiable independent of trusting whoever administers the log.

---

## 4. Multi-Framework Incident Mapping Schemas

### 4.1 Shared Incident Evidence Model

```json
{
  "incident_evidence_record": {
    "incident_id": "inc_2291",
    "discovered_at": "2026-06-21T10:00:00Z",
    "affected_system_id": "sys_fraud_detection_v3",
    "facts": {
      "description": "Model began incorrectly flagging transactions from a specific merchant category",
      "affected_transaction_count": 4200,
      "duration_minutes": 95,
      "personal_data_exposed": false,
      "service_disruption": true,
      "safety_or_fundamental_rights_risk": true
    }
  }
}
```

- **Principal-level note:** This single shared fact record is what each framework's assessment below reads from — the underlying facts don't change across frameworks, only the threshold and notification logic applied to them. Building four separate incident write-ups from scratch wastes time and risks inconsistency between them.

### 4.2 Per-Framework Reportability Assessment

```json
{
  "framework_assessments": [
    {
      "framework": "eu_ai_act",
      "threshold_metric": "risk_to_health_safety_fundamental_rights",
      "threshold_met": true,
      "reportable": true,
      "notification_deadline_hours": 15,
      "notify_to": "national_market_surveillance_authority"
    },
    {
      "framework": "dora",
      "threshold_metric": "client_impact_and_duration",
      "threshold_met": true,
      "reportable": true,
      "notification_deadline_hours": 4,
      "notify_to": "competent_financial_authority"
    },
    {
      "framework": "nis2",
      "threshold_metric": "service_disruption_significance",
      "threshold_met": false,
      "reportable": false,
      "notification_deadline_hours": null
    },
    {
      "framework": "gdpr",
      "threshold_metric": "risk_to_data_subjects",
      "threshold_met": false,
      "reportable": false,
      "notification_deadline_hours": null
    }
  ]
}
```

- **Principal-level note:** This is the concrete artifact behind "the same incident can legitimately clear one framework's threshold and not another's" — DORA's tighter deadline here means it governs the actual response timeline even though EU AI Act reportability is also true; track the *tightest* applicable deadline across all triggered frameworks, not each independently on its own clock.

### 4.3 Notification Submission Record

```json
{
  "regulatory_notification": {
    "incident_id": "inc_2291",
    "framework": "dora",
    "submitted_at": "2026-06-21T13:30:00Z",
    "deadline_was": "2026-06-21T14:00:00Z",
    "submission_type": "initial",
    "known_facts_complete": false,
    "follow_up_committed_by": "2026-06-23T00:00:00Z"
  }
}
```

- **Principal-level note:** `known_facts_complete: false` paired with a committed follow-up date is the correct pattern when root cause investigation is still ongoing at the deadline — regulators generally expect phased reporting, and missing the initial deadline while waiting for a complete picture is worse than filing an honestly incomplete initial report.

---

## 5. Conformity & Documentation Lifecycle

### 5.1 Risk Management File (Living Document Record)

```json
{
  "risk_management_file": {
    "system_id": "sys_fraud_detection_v3",
    "file_version": "5",
    "identified_risks": [
      { "risk": "false_positive_blocking_legitimate_transactions", "mitigation": "step_up_auth_for_moderate_confidence", "residual_risk": "low" }
    ],
    "continuous_monitoring_plan_ref": "monitoring_plan_v3",
    "last_updated_at": "2026-05-20T00:00:00Z",
    "update_trigger": "post_incident_review_inc_2291"
  }
}
```

- **Principal-level note:** `update_trigger` tied to a specific incident is what makes this genuinely a living document rather than a one-time filing — an incident that revealed an unforeseen failure mode is itself evidence the risk management file needs revision, and that update should be a required postmortem output, not a separate, easily-skipped follow-up task.

### 5.2 Technical Documentation Version Tied to Model Version

```json
{
  "technical_documentation": {
    "system_id": "sys_fraud_detection_v3",
    "model_version": "v14",
    "documentation_version": "v14_docs",
    "training_data_provenance_documented": true,
    "validation_methodology_documented": true,
    "known_limitations_documented": true,
    "audit_ready": true
  }
}
```

- **Principal-level note:** Documentation versions tied 1:1 to model versions in the same repository — documentation disconnected from the specific deployed model version is functionally useless during an audit, since you can't prove which documentation actually describes what's currently running.

### 5.3 Conformity Assessment Status

```json
{
  "conformity_assessment": {
    "system_id": "sys_fraud_detection_v3",
    "assessment_route": "self_assessment",
    "notified_body_required": false,
    "status": "completed",
    "ce_marking_issued": true,
    "declaration_of_conformity_ref": "doc_conformity_v14",
    "assessment_completed_before_deployment": true
  }
}
```

- **Principal-level note:** `assessment_completed_before_deployment: true` is the non-negotiable sequencing — high-risk systems don't deploy before conformity assessment completes, regardless of business pressure to ship. This field existing and being checked is what makes that policy enforceable rather than aspirational.

---

## 6. Complexity Reduction for Governance Specifically

| Degree of Freedom | Reduction Strategy |
|---|---|
| Classification review cadence | Fixed annual cycle plus a defined trigger list (material system change) — not ad hoc |
| Evidence formats across frameworks | One shared evidence model (Section 4.1), framework-specific views generated from it |
| Documentation ownership | Assigned to a role, not a person, with mandatory handover review on offboarding |
| Number of compliance tools/systems of record | One AI system inventory as the single source of truth, not parallel spreadsheets per team |

---

## 7. Decision Framework

1. Is there a complete, current AI system inventory, or are there AI systems running that haven't been catalogued and classified at all?
2. Is human oversight documented as an *operational record* of specific decisions, or only as a *policy statement* that oversight happens?
3. When an incident occurs, is there one shared evidence model feeding framework-specific assessments, or four separate reconstructions of the same facts?
4. Is documentation versioned 1:1 with the model version it describes, or could you not currently prove which documentation matches what's running in production right now?
5. Has conformity assessment ever been skipped or rushed to meet a deployment deadline — and if so, is that a known, accepted risk or an undiscovered gap?

**The governing test:** if a regulator asked for evidence right now, could it be produced from existing system records within hours — or would it require days of reconstruction, interviews, and hoping someone remembers what happened? The first is Level 3-4 governance; the second is Level 1, no matter how thorough the policy documents look.

---

## Companion Documents

Part of the Principal AI Engineer / FDE architecture series:

- `Agent_Orchestration_Architecture.md` — the provenance and audit metadata patterns this file expands into regulatory evidence schemas
- `RAG_Architecture_Deep_Dive.md` — provenance requirements for regulated RAG deployments referenced in Section 3
- `Model_Serving_Architecture_Deep_Dive.md` — data residency constraints on serving/routing decisions
- `Fine_Tuning_Workflow_Architecture.md` — the legal basis and data lineage requirements for training data referenced here
- `IAM_ZeroTrust_Agent_Architecture.md` — the access control evidence this governance model depends on
- `Observability_Evaluation_Architecture.md` — the underlying tracing and metrics infrastructure that generates much of the evidence referenced throughout this file
