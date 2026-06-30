# Fine-Tuning Workflow Architecture — Pipeline Patterns, Schemas & Evaluation Gates

**Principal AI Engineer / FDE Reference · Gunasekar Jabbala**

> Fine-tuning is the highest-maintenance-cost lever available, which means the architecture question isn't "how do I fine-tune" — it's "have I proven I actually need to." This document treats fine-tuning as a governed pipeline with explicit evaluation gates, the same discipline applied to model deployment and agent orchestration elsewhere in this series, because a fine-tuned model is a production artifact, not a one-time training run.

---

## Table of Contents

1. [The Fine-Tuning Maturity Model](#1-the-fine-tuning-maturity-model)
2. [Pipeline Architectural Patterns](#2-pipeline-architectural-patterns)
3. [Data Curation & Training Job Schemas](#3-data-curation--training-job-schemas)
4. [Evaluation Gate Contracts](#4-evaluation-gate-contracts)
5. [Deployment & Rollback Strategies](#5-deployment--rollback-strategies)
6. [Complexity Reduction for Fine-Tuning Specifically](#6-complexity-reduction-for-fine-tuning-specifically)
7. [Decision Framework](#7-decision-framework)

---

## 1. The Fine-Tuning Maturity Model

| Level | Name | Capabilities |
|---|---|---|
| **1** | Ad Hoc Fine-Tuning | Manual data prep, one-off training run, spot-check evaluation |
| **2** | Reproducible Pipeline | Versioned config (Axolotl-style YAML), tracked experiments, held-out evaluation set |
| **3** | Gated Production Pipeline | Automated regression suite, bias/safety evaluation, canary deployment of fine-tuned models |
| **4** | Continuous Fine-Tuning Platform | Production feedback loop into curation, automated re-evaluation on base model upgrades, full data lineage and compliance documentation |

The jump from Level 1 to Level 2 is mostly discipline (version everything). The jump to Level 3-4 is where most teams stall, because it requires building evaluation infrastructure that's often deprioritized until a fine-tuned model regresses in production with no one noticing for weeks.

---

## 2. Pipeline Architectural Patterns

### 2.1 The Decision Gate Before Fine-Tuning Begins

```
Problem -> Is this a knowledge gap, behavior gap, or format gap?
  Knowledge gap -> RAG
  Format/behavior gap, prompting-solvable -> Better prompting
  Format/behavior gap, prompting-insufficient at required reliability/volume -> Fine-tune
```

```json
{
  "fine_tuning_justification": {
    "problem_type": "behavior_gap",
    "prompting_attempted": true,
    "prompting_reliability_observed": 0.78,
    "required_reliability": 0.95,
    "volume_per_day": 50000,
    "decision": "fine_tune_justified",
    "alternative_considered": "few_shot_prompting"
  }
}
```

- **Principal-level note:** This gate should produce a written artifact, not just a verbal judgment call — when asked "why did you fine-tune instead of just prompting better," the strongest answer references the specific measured reliability gap that justified the decision, not a general sense that fine-tuning seemed appropriate.

### 2.2 QLoRA / Axolotl Pipeline Stages

```
Raw Data -> Curation & Validation -> Format Conversion -> Training Job (QLoRA) -> Checkpoint Evaluation -> Adapter Merge/Export -> Deployment Gate
```

```json
{
  "pipeline_run": {
    "run_id": "ft_run_221",
    "base_model": "model-base-v3",
    "method": "qlora",
    "lora_rank": 16,
    "lora_alpha": 32,
    "dataset_version": "curated_v4",
    "config_version": "axolotl_cfg_v7",
    "status": "checkpoint_evaluation"
  }
}
```

- **Principal-level note:** `dataset_version` and `config_version` being tracked as first-class, independently versioned fields is what makes a run reproducible. If asked "can you reproduce this exact fine-tuned model," the answer should be yes by re-running the pipeline against these two version pointers — not "I think I still have the files somewhere."

### 2.3 Multi-Task Adapter Architecture

```
Base Model (frozen)
  |-- LoRA Adapter A (task: classification)
  |-- LoRA Adapter B (task: summarization)
  `-- LoRA Adapter C (task: extraction)
```

```json
{
  "adapter_registry": {
    "base_model": "model-base-v3",
    "adapters": [
      { "adapter_id": "adapter_classification_v2", "task": "classification", "active": true },
      { "adapter_id": "adapter_summarization_v1", "task": "summarization", "active": true }
    ]
  }
}
```

- **Best for:** Serving multiple specialized behaviors from one base model without merging — adapters can be swapped per-request at serving time.
- **Trade-off:** More moving parts to version and evaluate independently; benefit is avoiding the cost of full separate fine-tunes per task.
- **Principal-level note:** Tie this directly to the model serving document's routing pattern (Section 2.3 there) — adapter selection at serving time is the fine-tuning-specific instance of the same task-based routing principle.

### 2.4 Continuous Fine-Tuning Feedback Loop

```
Production Inference -> Flagged/Sampled Outputs -> Human Review -> Curated Addition to Training Set -> Next Fine-Tuning Cycle
```

```json
{
  "feedback_loop_record": {
    "production_sample_id": "sample_8821",
    "flagged_reason": "low_confidence_output",
    "human_reviewed": true,
    "review_outcome": "add_to_training_set",
    "target_dataset_version": "curated_v5"
  }
}
```

- **Principal-level note:** This loop is what separates Level 4 from Level 3 — without it, every fine-tuning cycle starts from the same static dataset, and the model never improves on the specific failure patterns actually observed in production. The governance question to ask: who has authority to approve a flagged sample's addition to training data, since this is itself a data quality and bias control point.
---

## 3. Data Curation & Training Job Schemas

### 3.1 Dataset Version Manifest

```json
{
  "dataset_manifest": {
    "dataset_version": "curated_v4",
    "source_records": 18420,
    "after_deduplication": 16205,
    "after_quality_filter": 14890,
    "after_bias_review": 14890,
    "train_split": 13401,
    "eval_split": 1489,
    "created_at": "2026-05-15T00:00:00Z",
    "approved_by": "data_review_board",
    "provenance_sources": ["client_tickets_q1", "synthetic_augmentation_batch_3"]
  }
}
```

- **Principal-level note:** The funnel from `source_records` down to `eval_split` should be visible and auditable at every stage — when a fine-tuned model misbehaves, the first question is often "what was actually in the training data," and a dataset manifest with this granularity answers it without archaeology.

### 3.2 Individual Training Example Schema

```json
{
  "training_example_id": "ex_99102",
  "input": "...",
  "target_output": "...",
  "source": "human_curated",
  "quality_score": 0.91,
  "pii_scrubbed": true,
  "legal_basis_for_use": "legitimate_interest_documented",
  "added_to_dataset_version": "curated_v4"
}
```

- **Principal-level note:** `legal_basis_for_use` is not a paranoid addition — if training data includes personal data, GDPR requires a documented legal basis, and reconstructing that basis retroactively for thousands of examples after the fact is far harder than capturing it at curation time. *(See `AI_Governance_Compliance_Schemas.md` for the full regulatory treatment.)*

### 3.3 Training Job Configuration (Versioned)

```json
{
  "training_config": {
    "config_version": "axolotl_cfg_v7",
    "base_model": "model-base-v3",
    "method": "qlora",
    "lora_rank": 16,
    "lora_alpha": 32,
    "learning_rate": 0.0002,
    "epochs": 3,
    "batch_size": 8,
    "gradient_accumulation_steps": 4,
    "random_seed": 42
  }
}
```

- **Principal-level note:** `random_seed` being explicit and fixed is what makes a run's reproducibility claim actually testable — without it, "reproducible pipeline" is aspirational rather than verified.

### 3.4 Bias & Safety Pre-Training Audit

```json
{
  "bias_audit": {
    "dataset_version": "curated_v4",
    "demographic_dimensions_tested": ["none_applicable_to_domain"],
    "label_distribution_balanced": true,
    "known_limitations": "underrepresents_edge_case_X",
    "audit_completed_by": "domain_expert_reviewer",
    "approved_for_training": true
  }
}
```

- **Principal-level note:** Run this audit *before* training, not as an afterthought on the trained model's outputs — bias baked into training data is far more expensive to correct after a model has already been trained on it than to catch at the curation stage.

---

## 4. Evaluation Gate Contracts

### 4.1 Regression Suite Gate

```json
{
  "regression_gate": {
    "candidate_run_id": "ft_run_221",
    "regression_suite_version": "v9",
    "test_cases_total": 240,
    "test_cases_passed": 238,
    "previously_passing_now_failing": 0,
    "gate_status": "passed"
  }
}
```

- **Principal-level note:** `previously_passing_now_failing` is the single most important field — a high overall pass rate can hide the fact that the new fine-tune regressed on cases the previous version handled correctly. Treat any non-zero value here as a hard blocker regardless of overall pass rate.

### 4.2 Head-to-Head Comparison Against Base Model + Best Prompting

```json
{
  "comparison_eval": {
    "candidate": "fine_tuned_v221",
    "baseline": "base_model_v3_with_best_prompt",
    "eval_set_size": 500,
    "candidate_win_rate": 0.71,
    "baseline_win_rate": 0.19,
    "tie_rate": 0.10,
    "statistically_significant": true
  }
}
```

- **Principal-level note:** This comparison is the actual justification for having fine-tuned at all. Skipping it means you can't answer "how do you know fine-tuning was worth it" with anything other than an assertion — this schema is what turns that assertion into evidence.

### 4.3 Safety & Harmlessness Evaluation Gate

```json
{
  "safety_gate": {
    "candidate_run_id": "ft_run_221",
    "safety_suite_version": "v4",
    "harmful_output_rate": 0.002,
    "harmful_output_threshold": 0.005,
    "safety_regression_vs_base": false,
    "gate_status": "passed"
  }
}
```

- **Principal-level note:** Explicitly check for safety *regression* relative to the base model, not just an absolute threshold — fine-tuning for task performance can inadvertently erode safety properties the base model had, and that's a distinct failure mode from simply "not safe enough" in absolute terms.

### 4.4 Deployment Readiness Gate (Aggregating All Prior Gates)

```json
{
  "deployment_readiness": {
    "candidate_run_id": "ft_run_221",
    "regression_gate": "passed",
    "comparison_eval": "passed",
    "safety_gate": "passed",
    "bias_audit": "passed",
    "overall_status": "approved_for_canary",
    "approved_by": "ml_review_board",
    "approved_at": "2026-06-10T00:00:00Z"
  }
}
```

---

## 5. Deployment & Rollback Strategies

### 5.1 Fine-Tuned Model Canary Deployment

This reuses the model serving document's canary pattern directly — a fine-tuned model is just another model version from the serving layer's perspective, and should go through the identical rollout discipline (Section 2.5 there), not a separate, less rigorous path just because it came from a fine-tuning pipeline rather than a vendor release.

### 5.2 Adapter Hot-Swap Rollback

```json
{
  "adapter_rollback": {
    "tenant_or_workflow": "classification_task",
    "active_adapter": "adapter_classification_v2",
    "rollback_target": "adapter_classification_v1",
    "rollback_trigger": "quality_regression_detected",
    "rollback_time_seconds": 4
  }
}
```

- **Principal-level note:** This is the specific advantage of the adapter architecture (Section 2.3) over a fully merged fine-tuned model — rollback is a fast pointer swap, not a redeploy. Name this explicitly as a reason to prefer adapters over merged weights when rollback speed matters.

---

## 6. Complexity Reduction for Fine-Tuning Specifically

| Degree of Freedom | Reduction Strategy |
|---|---|
| Number of active fine-tuning experiments | Cap concurrent in-flight experiments; finish and gate one before starting unrelated new ones |
| Hyperparameter search space | Start with documented-good defaults; only sweep when a specific evaluation gap justifies the search cost |
| Number of adapters in production | One adapter per genuinely distinct task — resist creating narrow adapters for marginal variations |
| Dataset versions in circulation | One active "current" version at a time, with a clear, documented promotion process from candidate to current |

---

## 7. Decision Framework

1. Is this actually a fine-tuning problem, or would RAG or better prompting solve it more cheaply? (Section 2.1 — never skip this gate.)
2. Has the candidate model been compared head-to-head against the best achievable prompting baseline, not just against "no fine-tuning at all"?
3. Does the regression suite check for regressions on previously-passing cases specifically, not just an aggregate pass rate?
4. Is there a safety/bias gate that runs before deployment, not just a task-performance gate?
5. Is rollback fast (adapter swap) or slow (full redeploy) — and does that speed match how much you trust this specific fine-tuning pipeline's reliability?

**The governing test:** a fine-tuned model should be exactly as reproducible, versioned, and gated as any other production deployment — the fact that training is statistically noisy doesn't excuse the surrounding pipeline from being deterministic and auditable.

---

## Companion Documents

Part of the Principal AI Engineer / FDE architecture series:

- `Agent_Orchestration_Architecture.md` — the evaluation and confidence-gating principles applied here to training pipelines
- `RAG_Architecture_Deep_Dive.md` — the alternative to fine-tuning for knowledge-gap problems
- `Model_Serving_Architecture_Deep_Dive.md` — the canary/rollback infrastructure fine-tuned models deploy through
- `IAM_ZeroTrust_Agent_Architecture.md` — access control over who can approve dataset changes and deployment gates
- `AI_Governance_Compliance_Schemas.md` — the legal basis and data lineage requirements referenced in Section 3.2
- `Observability_Evaluation_Architecture.md` — the full evaluation framework underlying every gate in Section 4
