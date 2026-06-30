# Observability & Evaluation Architecture — Tracing Schemas, Eval Frameworks & Cost Dashboards

**Principal AI Engineer / FDE Reference · Gunasekar Jabbala**

> "You cannot orchestrate what you cannot measure" is the governing principle of this entire document series, and this file is where that principle becomes concrete infrastructure. Every schema referenced across the other six files — provenance, evaluation, confidence, audit metadata — ultimately depends on the tracing and metrics foundation built here. This is the layer that turns "it seems to be working" into "here is the evidence it's working, and here's exactly where it breaks."

---

## Table of Contents

1. [The Observability Maturity Model](#1-the-observability-maturity-model)
2. [Distributed Tracing Schemas](#2-distributed-tracing-schemas)
3. [Evaluation Framework Contracts](#3-evaluation-framework-contracts)
4. [Cost & Latency Dashboard Schemas](#4-cost--latency-dashboard-schemas)
5. [Failure Analysis & Chaos Engineering Patterns](#5-failure-analysis--chaos-engineering-patterns)
6. [Complexity Reduction for Observability Specifically](#6-complexity-reduction-for-observability-specifically)
7. [Decision Framework](#7-decision-framework)

---

## 1. The Observability Maturity Model

| Level | Name | Capabilities |
|---|---|---|
| **1** | Basic Logging | Free-text logs, no correlation across components, manual log searching during incidents |
| **2** | Structured Tracing | Correlated trace IDs across a workflow, basic dashboards (latency, error rate) |
| **3** | Full-Stack Observability | End-to-end decision reconstruction, cost-per-task tracking, automated anomaly detection |
| **4** | Continuous Evaluation Platform | Production-integrated evaluation (not just pre-deployment), regression detection in real time, chaos engineering as a routine practice |

Most teams treat observability as a debugging convenience (Level 1-2) rather than a compliance and forensic asset (Level 3-4). The reframe that matters in an interview: observability infrastructure is what makes every other governance and reliability claim in this document series actually verifiable rather than asserted.

---

## 2. Distributed Tracing Schemas

### 2.1 OpenTelemetry-Aligned Trace Envelope

```json
{
  "trace": {
    "trace_id": "otel_trace_abc123",
    "root_span_id": "span_001",
    "workflow_id": "wf_001",
    "spans": [
      { "span_id": "span_001", "parent_span_id": null, "operation": "supervisor_route", "start_ms": 0, "duration_ms": 12 },
      { "span_id": "span_002", "parent_span_id": "span_001", "operation": "research_agent_retrieve", "start_ms": 12, "duration_ms": 340 },
      { "span_id": "span_003", "parent_span_id": "span_001", "operation": "generation_call", "start_ms": 352, "duration_ms": 1800 }
    ]
  }
}
```

- **Principal-level note:** This directly reuses the `trace_id`/`span_id` fields from the orchestration document's mandatory message envelope (Section 4.1 there) — tracing isn't a separate system bolted on afterward, it's a property every inter-agent message carries from the start. If trace propagation isn't designed into the message envelope from day one, retrofitting it later means re-instrumenting every component.

### 2.2 Full Decision Reconstruction Record

```json
{
  "decision_reconstruction": {
    "request_id": "req_9981",
    "trace_id": "otel_trace_abc123",
    "input": "...",
    "retrieved_context": [{ "chunk_id": "chunk_4521", "relevance_score": 0.91 }],
    "reasoning_summary": "Classified as high-risk based on velocity and geographic anomaly signals",
    "tool_calls": [{ "tool": "crm_api", "result_summary": "customer flagged previously" }],
    "final_output": "...",
    "model_version": "v14",
    "human_reviewed": false
  }
}
```

- **Principal-level note:** This is the artifact that answers "what did the model actually see" for any given bad output — the question that comes up in nearly every production AI incident postmortem across every other file in this series. If you can't reconstruct this record for a specific request, you don't have production-grade observability regardless of what dashboards exist.

### 2.3 Cross-Agent Interaction Graph

```json
{
  "interaction_graph": {
    "workflow_id": "wf_001",
    "nodes": ["supervisor", "research_agent", "review_agent"],
    "edges": [
      { "from": "supervisor", "to": "research_agent", "message_count": 3 },
      { "from": "research_agent", "to": "review_agent", "message_count": 1 }
    ],
    "total_handoffs": 4,
    "max_depth": 2
  }
}
```

- **Principal-level note:** Visualizing the actual interaction graph (not just individual traces) is what surfaces emergent complexity — a workflow that looks simple in its design documentation but has accumulated many more handoffs than intended in practice. This is the concrete tool for catching the orchestration document's "degrees of freedom" creeping upward over time without anyone deciding that should happen.
---

## 3. Evaluation Framework Contracts

### 3.1 Continuous Production Evaluation Record

```json
{
  "production_eval": {
    "sample_id": "sample_5521",
    "evaluation_method": "llm_as_judge",
    "judge_model": "eval-model-v2",
    "dimensions": { "groundedness": 0.91, "relevance": 0.88, "safety": 1.0 },
    "human_reviewed": false,
    "flagged_for_human_review": false,
    "sampled_at": "2026-06-21T12:00:00Z"
  }
}
```

- **Principal-level note:** Evaluation that only happens before deployment is Level 2-3. Level 4 means continuously sampling production traffic for ongoing evaluation, since real-world query distribution and edge cases reliably diverge from whatever pre-deployment evaluation set was built before launch.

### 3.2 Regression Detection Trigger

```json
{
  "regression_alert": {
    "metric": "groundedness_score_rolling_avg",
    "baseline_value": 0.91,
    "current_value": 0.79,
    "threshold_breach_percentage": 13.2,
    "detected_at": "2026-06-21T12:00:00Z",
    "likely_cause_candidates": ["model_version_change", "retrieval_index_update", "upstream_data_drift"],
    "auto_action": "alert_only_no_auto_rollback"
  }
}
```

- **Principal-level note:** `likely_cause_candidates` being a structured list (not a single guess) reflects the reality that regression diagnosis usually starts with several plausible hypotheses to rule in or out systematically — exactly the elimination-based diagnostic approach valued throughout the MongoDB internals and FDE scenario files in this series.

### 3.3 Adversarial / Red-Team Evaluation Record

```json
{
  "adversarial_eval": {
    "test_suite": "prompt_injection_v3",
    "test_cases_run": 120,
    "successful_injections": 2,
    "injection_vectors": ["instruction_override_via_retrieved_document"],
    "severity": "medium",
    "remediation_status": "in_progress"
  }
}
```

- **Principal-level note:** "Never trust demos" means deliberately breaking your own agents before an adversary does — this record format is what makes that practice auditable and trackable over time, rather than a one-off pentest exercise whose findings get fixed and then forgotten as a recurring practice.

### 3.4 Human Override Rate Tracking

```json
{
  "human_override_metric": {
    "time_window": "2026-06-01_to_2026-06-21",
    "total_ai_decisions": 84200,
    "human_overrides": 1260,
    "override_rate": 0.015,
    "override_rate_trend": "increasing",
    "top_override_reasons": ["edge_case_not_in_training_data", "context_human_had_that_model_lacked"]
  }
}
```

- **Principal-level note:** An increasing override rate trend is an early warning signal worth surfacing proactively — it often means either the underlying task distribution is shifting (drift) or the model/system hasn't kept pace with a changing environment, and catching this trend early is cheaper than waiting for it to become a visible quality incident.

---

## 4. Cost & Latency Dashboard Schemas

### 4.1 Cost Attribution Record

```json
{
  "cost_record": {
    "request_id": "req_9981",
    "tenant_id": "tenant_acme",
    "workflow_id": "wf_001",
    "cost_breakdown": {
      "embedding_calls": 0.0008,
      "retrieval_compute": 0.0001,
      "generation_call": 0.0041,
      "tool_calls": 0.0002
    },
    "total_cost_usd": 0.0052
  }
}
```

- **Principal-level note:** Breaking cost down by component (not just a single total) is what makes cost optimization targeted rather than guesswork — if generation dominates the cost breakdown, optimizing retrieval infrastructure further is wasted effort, and this schema makes that immediately visible.

### 4.2 Latency Distribution (Not Just Average)

```json
{
  "latency_distribution": {
    "metric": "end_to_end_request_latency_ms",
    "time_window": "last_1_hour",
    "p50": 420,
    "p95": 1100,
    "p99": 3200,
    "p99_9": 8500,
    "sample_count": 14200
  }
}
```

- **Principal-level note:** A system with excellent average latency can still produce a meaningfully bad experience for a non-trivial fraction of users if tail latency (p99, p99.9) is poor — tracking the full distribution, not just the average, is what surfaces this; average-only monitoring is a Level 1-2 gap that hides real user-facing problems.

### 4.3 Token Consumption & Efficiency Dashboard Record

```json
{
  "token_efficiency": {
    "workflow_id": "wf_001",
    "time_window": "last_24_hours",
    "total_tokens_consumed": 4200000,
    "tokens_per_successful_task": 1850,
    "tokens_per_failed_task": 3100,
    "efficiency_trend": "stable"
  }
}
```

- **Principal-level note:** `tokens_per_failed_task` being notably higher than `tokens_per_successful_task` is a specific, actionable signal — it often means failing tasks are retrying or looping before ultimately failing, burning cost on attempts that don't succeed, which is exactly the failure mode the orchestration document's bounded-retry strategies are meant to prevent.

---

## 5. Failure Analysis & Chaos Engineering Patterns

### 5.1 Chaos Experiment Record

```json
{
  "chaos_experiment": {
    "experiment_id": "chaos_221",
    "hypothesis": "System gracefully degrades when the embedding service times out",
    "injected_failure": "embedding_service_timeout",
    "injected_at": "2026-06-21T12:00:00Z",
    "observed_behavior": "Fallback to cached embeddings triggered correctly; no user-facing error",
    "hypothesis_confirmed": true
  }
}
```

- **Principal-level note:** A chaos experiment with an explicit hypothesis (stated before running it) is meaningfully different from randomly breaking things and seeing what happens — the hypothesis-driven version produces a clear pass/fail result that either confirms a resilience assumption or surfaces a gap, while undirected chaos testing often just produces noise without a clear actionable conclusion.

### 5.2 Mean Time to Recovery (MTTR) Tracking

```json
{
  "incident_mttr": {
    "incident_id": "inc_2291",
    "detected_at": "2026-06-21T10:00:00Z",
    "contained_at": "2026-06-21T10:08:00Z",
    "resolved_at": "2026-06-21T11:35:00Z",
    "time_to_detect_minutes": 3,
    "time_to_contain_minutes": 5,
    "time_to_resolve_minutes": 87
  }
}
```

- **Principal-level note:** Breaking MTTR into detect/contain/resolve phases separately (not one aggregate number) reveals where the actual bottleneck is — a fast detection and containment with a slow resolve points to a different improvement priority than a slow detection with fast containment and resolve once found.

### 5.3 Postmortem Action Item Tracking

```json
{
  "postmortem_action_item": {
    "incident_id": "inc_2291",
    "action": "Add input validation for merchant category codes from this specific upstream source",
    "owner": "engineer_user_3301",
    "due_date": "2026-07-01",
    "status": "in_progress",
    "would_have_prevented_this_incident": true
  }
}
```

- **Principal-level note:** Tracking `would_have_prevented_this_incident` against each *previous* postmortem's action items, checked at the time of a *new* incident, is the single highest-value habit referenced across this entire document series — it's what turns postmortems from a documentation exercise into genuine continuous improvement, by revealing whether past commitments were actually followed through on.

---

## 6. Complexity Reduction for Observability Specifically

| Degree of Freedom | Reduction Strategy |
|---|---|
| Logging detail level | Tiered: full trace for a sampled percentage, summary-level for the rest, with ability to retroactively enable full tracing for a flagged session |
| Number of dashboards/metrics tracked | A small set of governing metrics per layer (cost, latency distribution, override rate, groundedness) rather than tracking everything technically loggable |
| Alert thresholds | Calibrated against actual production baseline data, reviewed periodically — not arbitrary round numbers picked at design time |
| Evaluation cadence | Continuous sampling in production, not only pre-deployment, but sampling rate tuned to cost rather than evaluating every single request |

---

## 7. Decision Framework

1. For any given bad output, can you reconstruct exactly what the system saw and decided — input, retrieved context, reasoning summary, tool calls, output — or only the final result?
2. Are you tracking latency and quality as *distributions* (p50/p95/p99, full eval score spread) or only as single aggregate averages that can hide real tail-case problems?
3. Is evaluation happening continuously against live production traffic, or only as a one-time gate before deployment?
4. Do your chaos experiments test specific, stated hypotheses about resilience, or is failure injection undirected and hard to draw clear conclusions from?
5. When you look back at your last several postmortems, would their action items — if actually implemented — have prevented your most recent incident? If you don't know the answer, that's itself the finding.

**The governing test, closing this entire six-part architecture series:** every claim made anywhere in these documents — about reliability, about cost-efficiency, about security boundaries holding, about compliance evidence existing — should be backed by something measurable and reconstructable in this file's infrastructure. An architecture document describes intent; observability infrastructure is what proves the intent was actually realized in production.

---

## Companion Documents

This file completes the Principal AI Engineer / FDE architecture series:

- `Agent_Orchestration_Architecture.md` — the trace/correlation ID fields this file's tracing schemas extend
- `RAG_Architecture_Deep_Dive.md` — the groundedness and retrieval-quality evaluation referenced in Section 3
- `Model_Serving_Architecture_Deep_Dive.md` — the cost and latency dashboards built on this file's schemas
- `Fine_Tuning_Workflow_Architecture.md` — the evaluation gates this file's continuous evaluation infrastructure powers
- `IAM_ZeroTrust_Agent_Architecture.md` — the behavioral baseline and anomaly detection infrastructure referenced in Section 2.4 of that file
- `AI_Governance_Compliance_Schemas.md` — the evidence-generation infrastructure this file underpins for regulatory purposes
