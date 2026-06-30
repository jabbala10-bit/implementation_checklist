# IAM & Zero-Trust Agent Architecture — Identity, Credentials & Policy Enforcement

**Principal AI Engineer / FDE Reference · Gunasekar Jabbala**

> The governing question throughout this document, inherited from the orchestration file: **"Should this agent be allowed to do this action?"** — not "can this agent do this action?" Capability checks answer the second question; policy decisions answer the first. Most agent security failures happen because a system only ever asked the first question.

---

## Table of Contents

1. [The Agent Identity Maturity Model](#1-the-agent-identity-maturity-model)
2. [Agent Identity Patterns](#2-agent-identity-patterns)
3. [Credential Lifecycle Schemas](#3-credential-lifecycle-schemas)
4. [Policy Enforcement Contracts](#4-policy-enforcement-contracts)
5. [Zero-Trust Verification Strategies](#5-zero-trust-verification-strategies)
6. [Complexity Reduction for Agent IAM Specifically](#6-complexity-reduction-for-agent-iam-specifically)
7. [Decision Framework](#7-decision-framework)

---

## 1. The Agent Identity Maturity Model

| Level | Name | Capabilities |
|---|---|---|
| **1** | Borrowed Identity | Agent reuses the initiating human's or a shared service credential |
| **2** | Distinct Agent Identity | Each agent has its own identity, but with broad, standing permissions |
| **3** | Scoped, Time-Bound Access | Least-privilege scoping, short-lived credentials, per-task issuance |
| **4** | Continuous Zero-Trust Verification | Re-evaluation at every significant action (not just session start), behavioral baselining, intent-aware policy decisions |

Level 1 is the most common starting point and the most common root cause cited in agent-related security incidents — an agent acting under a human's identity means every audit log entry is ambiguous about who (or what) actually took the action.

---

## 2. Agent Identity Patterns

### 2.1 First-Class Agent Identity (Distinct from Initiating Human)

```json
{
  "agent_identity": {
    "agent_id": "agent_research_007",
    "agent_type": "research_agent",
    "identity_class": "non_human_service_principal",
    "created_by_human": "user_4421",
    "created_at": "2026-06-21T10:00:00Z",
    "standing_permissions": [],
    "max_session_duration_seconds": 3600
  }
}
```

- **Principal-level note:** `standing_permissions: []` is deliberate — Level 3+ identity carries no permissions by default; everything is granted per-task (Section 3). `created_by_human` preserves accountability lineage without conflating the agent's actions with the human's own identity in the audit log.

### 2.2 Agent-to-Agent Authentication

```json
{
  "agent_auth_request": {
    "calling_agent_id": "agent_supervisor_001",
    "target_agent_id": "agent_executor_004",
    "auth_method": "mutual_tls",
    "calling_agent_cert_fingerprint": "a1:b2:c3",
    "request_signed": true,
    "signature_valid": true
  }
}
```

- **Best for:** Any agent-to-agent call crossing a trust boundary — which, in a zero-trust model, is every call, not just ones crossing network boundaries.
- **Principal-level note:** Mutual TLS gives a stronger cryptographic identity guarantee than a bearer token alone. Use it for service-to-service agent communication wherever infrastructure supports it, and explain the specific guarantee it adds (the caller cryptographically proves identity, not just presents a possibly-stolen token) when asked why it matters over a simpler API key.

### 2.3 Capability-Scoped Tool Access (Mapped from Orchestration Schema)

```json
{
  "agent_tool_grant": {
    "agent_id": "agent_research_007",
    "granted_tools": ["search", "document_read"],
    "explicitly_denied_tools": ["crm_write", "payment_execute"],
    "grant_expires_at": "2026-06-21T11:00:00Z",
    "grant_reason": "task_9981_research_phase"
  }
}
```

- **Principal-level note:** This is the IAM-layer expansion of the orchestration document's Policy Enforcement Pattern (Section 4.18 there) — same principle, but now with full credential lifecycle fields (expiry, reason, explicit denial list) appropriate for an identity and access management system rather than a lightweight in-workflow policy object.

### 2.4 Behavioral Baseline & Anomaly Detection

```json
{
  "agent_behavior_baseline": {
    "agent_id": "agent_research_007",
    "baseline_metric": "tool_calls_per_session",
    "baseline_mean": 4.2,
    "baseline_stddev": 1.1,
    "current_session_value": 19,
    "anomaly_detected": true,
    "action": "flag_for_review_pause_session"
  }
}
```

- **Principal-level note:** This is the same user-behavior-analytics technique applied to human privileged accounts, transposed to agent identities. An agent suddenly calling tools far outside its normal pattern is exactly as suspicious as a human account suddenly accessing systems it's never touched before — apply the same detection discipline rather than treating agent monitoring as a separate, less mature discipline.
---

## 3. Credential Lifecycle Schemas

### 3.1 Just-in-Time Credential Issuance

```json
{
  "jit_credential": {
    "credential_id": "cred_5521",
    "agent_id": "agent_executor_004",
    "issued_for_task_id": "task_9981",
    "issued_at": "2026-06-21T12:00:00Z",
    "expires_at": "2026-06-21T12:30:00Z",
    "scope": ["crm_write:single_record"],
    "revocable": true,
    "auto_revoke_on_task_completion": true
  }
}
```

- **Principal-level note:** `auto_revoke_on_task_completion` matters as much as the time-bound expiry — a credential that outlives its task is latent risk even within its expiry window. Revoke proactively on completion rather than relying purely on the TTL to eventually expire it.

### 3.2 Credential Rotation Record

```json
{
  "rotation_event": {
    "credential_family": "agent_research_service_account",
    "previous_credential_id": "cred_5400",
    "new_credential_id": "cred_5521",
    "rotation_trigger": "scheduled",
    "rotation_interval_hours": 24,
    "old_credential_revoked_at": "2026-06-21T12:00:05Z"
  }
}
```

- **Principal-level note:** Automate rotation on a fixed short interval rather than relying on manual discipline. The gap between `new_credential_id` issuance and `old_credential_revoked_at` should be seconds, not a separate manual cleanup step that can be forgotten.

### 3.3 Break-Glass Emergency Access Record

```json
{
  "break_glass_access": {
    "access_id": "bg_0012",
    "requested_by": "user_4421",
    "justification": "production_incident_INC-2291",
    "approved_by": "security_lead_user_1102",
    "scope": ["production_db:read_only"],
    "auto_expires_at": "2026-06-21T14:00:00Z",
    "post_use_review_required": true,
    "post_use_review_completed": false
  }
}
```

- **Principal-level note:** Break-glass paths are necessary but need to be *more* heavily audited than normal access, not less — the `post_use_review_required` flag enforces that every emergency access gets a mandatory retrospective look, since the normal pre-approval rigor was deliberately bypassed to enable speed during the incident.

### 3.4 Secrets-in-Prompt Prevention Contract

```json
{
  "secret_scan_result": {
    "context_id": "ctx_8821",
    "scanned_for": "agent_prompt_and_tool_output",
    "secrets_detected": 0,
    "scan_passed": true,
    "scan_method": "pattern_match_plus_entropy_analysis"
  }
}
```

- **Principal-level note:** Secrets should never enter the prompt or context window in the first place — injected at runtime via a vault with short-lived tokens, fully outside the agent's visibility. This scan is a defense-in-depth backstop, not the primary control; if this scan is ever what catches a secret, something upstream already failed.

---

## 4. Policy Enforcement Contracts

### 4.1 Centralized Policy Decision Point (PDP) Request

```json
{
  "policy_decision_request": {
    "request_id": "pdp_req_3321",
    "agent_id": "agent_executor_004",
    "requested_action": "delete_customer_record",
    "resource": "customer_8821",
    "context": { "task_risk_tier": "high", "human_approval_present": false }
  }
}
```

```json
{
  "policy_decision_response": {
    "request_id": "pdp_req_3321",
    "decision": "deny",
    "reason": "high_risk_action_requires_human_approval",
    "policy_id": "policy_delete_requires_approval_v3"
  }
}
```

- **Principal-level note:** Policy decisions should be centralized and externalized from agent logic — an agent should call a policy decision point and receive a decision, not evaluate its own permissions internally. This is the architectural enforcement of "agents should not decide their own permissions" from the orchestration document's Section 4.18.

### 4.2 Segregation of Duties Enforcement

```json
{
  "segregation_check": {
    "agent_id": "agent_finance_001",
    "action_requested": "approve_payment",
    "same_agent_previously_initiated": true,
    "initiating_action": "create_payment_request",
    "decision": "deny",
    "reason": "segregation_of_duties_violation"
  }
}
```

- **Principal-level note:** Apply the same segregation-of-duties principle that prevents a single human from having conflicting authority — an agent that can both initiate and approve the same class of action violates segregation of duties regardless of being non-human. This is a frequently missed control in agent system designs that otherwise look security-conscious.

### 4.3 Data Classification-Aware Access Policy

```json
{
  "data_access_policy": {
    "agent_id": "agent_research_007",
    "requested_resource_classification": "confidential",
    "agent_clearance_level": "internal_only",
    "decision": "deny",
    "reason": "insufficient_clearance_for_classification"
  }
}
```

- **Principal-level note:** Data classification should be a first-class attribute checked at the policy layer, not something agents are trusted to respect voluntarily based on prompt instructions — prompt-based restrictions are not a security control, they're a suggestion a sufficiently adversarial input can override.

---

## 5. Zero-Trust Verification Strategies

### 5.1 Continuous Re-Verification (Not Just Session Start)

```json
{
  "continuous_verification": {
    "agent_id": "agent_executor_004",
    "session_id": "sess_7711",
    "verification_checkpoints": [
      { "action_number": 1, "verified_at": "12:00:00", "result": "pass" },
      { "action_number": 8, "verified_at": "12:04:00", "result": "pass" },
      { "action_number": 15, "verified_at": "12:09:00", "result": "fail_context_changed" }
    ]
  }
}
```

- **Principal-level note:** A one-time authentication check at session start is a Level 2 control. Zero-trust maturity means re-evaluating context and risk signals at each significant action — an agent's behavior or the surrounding context can shift mid-session in ways a single upfront check would miss entirely.

### 5.2 Micro-Segmentation for Agent-to-Service Communication

```json
{
  "network_policy": {
    "agent_type": "research_agent",
    "allowed_destinations": ["search_service", "document_store"],
    "denied_destinations": ["payment_service", "production_database"],
    "enforcement_layer": "service_mesh"
  }
}
```

- **Principal-level note:** An agent compromised within one service shouldn't be able to laterally reach unrelated services — apply the same blast-radius thinking to agent-to-service communication as to human-to-service access. `enforcement_layer: service_mesh` matters because enforcement at the infrastructure layer survives even if the agent's own logic is compromised; application-layer-only enforcement doesn't.

### 5.3 Intent Verification Before High-Risk Action Execution

```json
{
  "intent_verification": {
    "agent_id": "agent_executor_004",
    "planned_action": "bulk_delete_records",
    "record_count": 4200,
    "typical_action_scope": "single_record",
    "deviation_flagged": true,
    "action_held_pending_review": true
  }
}
```

- **Principal-level note:** This is the forward-looking evolution this document anticipates — policy engines evaluating not just identity and access but the agent's actual planned action and reasoning before execution. Today's zero-trust models mostly evaluate access; naming this gap and how you'd start closing it is a strong forward-thinking answer for a Principal-level systems design conversation.

---

## 6. Complexity Reduction for Agent IAM Specifically

| Degree of Freedom | Reduction Strategy |
|---|---|
| Standing permissions per agent | Default to zero; everything granted just-in-time per task |
| Credential lifetime | Shortest viable TTL tied to task duration, not a generic long-lived default |
| Policy decision points | One centralized PDP, not per-agent embedded permission logic |
| Trust boundaries crossed without re-verification | Zero — every boundary crossing re-verifies, even within an "internal" network |

---

## 7. Decision Framework

1. Does this agent have its own distinct identity, or is it borrowing a human's or a shared service account's credentials?
2. Are permissions granted just-in-time and scoped to the specific task, or does the agent hold standing access "just in case"?
3. Is policy evaluation centralized and externalized, or can the agent's own logic effectively decide its own permissions?
4. Does verification happen continuously through a session, or only once at session start?
5. If this agent's credentials were compromised right now, what's the actual blast radius — and is that radius something you deliberately designed, or just whatever resulted from how access happened to be granted?

**The governing test:** every agent action should be attributable to a specific, scoped, revocable identity — not to "the agent system" generically, and not to the human who happened to trigger it. If you can't answer "exactly what could this specific agent have done in this specific window" precisely, the identity model isn't mature enough yet.

---

## Companion Documents

Part of the Principal AI Engineer / FDE architecture series:

- `Agent_Orchestration_Architecture.md` — the policy enforcement and tool access patterns this file expands into full IAM schemas
- `RAG_Architecture_Deep_Dive.md` — the access-control-aware retrieval this identity model enforces
- `Model_Serving_Architecture_Deep_Dive.md` — tenant identity feeding the multi-tenancy resource isolation contracts
- `Fine_Tuning_Workflow_Architecture.md` — access control over training data and deployment gate approvals
- `AI_Governance_Compliance_Schemas.md` — the regulatory access-control and audit requirements this identity model satisfies
- `Observability_Evaluation_Architecture.md` — the behavioral baseline and anomaly detection infrastructure referenced in Section 2.4
