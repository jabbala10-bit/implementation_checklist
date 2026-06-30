# Agent Orchestration Architecture — Patterns, Schemas & Complexity Reduction

**Principal AI Engineer / FDE Reference · Gunasekar Jabbala**

> Agent orchestration is distributed systems engineering with probabilistic components. The transition from "building agents" to "mastering orchestration" happens when you stop asking *"how do I call an LLM?"* and start asking *"how do I coordinate autonomous systems safely, reliably, and economically at scale?"*
>
> Most engineers approach agents from the prompt layer. Experts think in systems engineering — state ownership, failure modes, blast radius, and cost per decision. This document treats agent interactions as well-defined contracts, not free-form conversations.

---

## Table of Contents

1. [The Agent Orchestration Maturity Model](#1-the-agent-orchestration-maturity-model)
2. [Core Mental Models](#2-core-mental-models)
3. [Architectural Patterns](#3-architectural-patterns)
4. [Inter-Agent Schema Catalog](#4-inter-agent-schema-catalog)
5. [Complexity & Dynamics Reduction Strategies](#5-complexity--dynamics-reduction-strategies)
6. [Pattern-Specific Recommendations](#6-pattern-specific-recommendations)
7. [Economics, Observability & Security](#7-economics-observability--security)
8. [The Enterprise Agent Stack](#8-the-enterprise-agent-stack)
9. [Decision Framework & Closing Heuristics](#9-decision-framework--closing-heuristics)

---

## 1. The Agent Orchestration Maturity Model

Most teams operate between Levels 1 and 2. Expert orchestrators operate at Levels 3 and 4.

| Level | Name | Capabilities |
|---|---|---|
| **1** | Single Agent | Prompt engineering, tool calling, basic memory, RAG integration, function execution |
| **2** | Multi-Agent Workflows | Planner → Executor → Reviewer, agent handoffs, shared memory, context management, failure handling |
| **3** | Production Orchestration | State machines, workflow engines, human-in-the-loop, observability, cost optimization, security boundaries |
| **4** | Autonomous Systems | Dynamic agent creation, agent marketplaces, self-improving workflows, multi-model routing, policy-driven execution, governance |

**Interview framing:** when asked "what level is your orchestration experience," answer in terms of which layer you've actually built production controls for — not which framework you've used. A Level 2 implementation using a sophisticated framework is still Level 2 if it lacks state machines, observability, and security boundaries.

---

## 2. Core Mental Models

**Think like an operating system designer, not an AI engineer.**

Every orchestration problem has six layers. If any layer is weak, your agents become expensive demos.

| Layer | Core Question |
|---|---|
| Goals | What outcome is required? |
| Planning | How is work decomposed? |
| Execution | Which agent performs which task? |
| Coordination | How do agents communicate? |
| Governance | What constraints exist? |
| Observability | How do we measure success? |

**Become obsessed with state, not prompts.** Experts spend more time designing state than prompts. Define explicitly:
- What agents know
- What agents forget
- What agents share
- What agents can modify

Ask continuously:
- Is memory ephemeral or persistent?
- Who owns the source of truth?
- How is context refreshed?
- How is stale information detected?

**Ask "should," not "can."** Security and governance separate experts from enthusiasts: ask *"should this agent be allowed to do this action?"* — not *"can this agent do this action?"* The first is a policy decision; the second is a capability check. Conflating them is how over-permissioned agents happen.

---

## 3. Architectural Patterns

### 3.1 Supervisor Pattern

A coordinator agent delegates tasks to specialists.

```
User
  ↓
Supervisor
  ├── Research Agent
  ├── Coding Agent
  ├── QA Agent
  └── Security Agent
```

- **Best for:** Enterprise copilots, customer support, research assistants.
- **Trade-off:** Simple to control; supervisor becomes a bottleneck and a single point of failure.
- **Principal-level note:** The bottleneck is a feature, not just a cost — it's your single audit and policy enforcement point. Don't "fix" the bottleneck by allowing agents to bypass the supervisor for speed; fix it by making the supervisor fast and stateless enough to scale horizontally.

### 3.2 Planner-Executor Pattern

```
Goal -> Planner -> Task List -> Executors
```

- **Best for:** Complex workflows, long-running tasks, dynamic objectives.
- **Trade-off:** Higher latency; better reliability.
- **Principal-level note:** The planner's output should be a reviewable, versioned artifact (a task list with explicit dependencies) — not an implicit decision the executor has to infer. This is what makes the pattern replayable and auditable.

### 3.3 Event-Driven Agents

Agents react to events instead of conversations.

```
Event Bus
   ↓
Agent A -> Agent B -> Agent C
```

- **Best for:** Enterprise automation, IT operations, incident response, IAM workflows.
- **Learn:** Event sourcing, Kafka, Pub/Sub, CQRS.
- **Principal-level note:** This pattern is especially valuable if your background includes IAM or security operations — event-driven agent architectures map directly onto SIEM/SOAR automation patterns you likely already understand. The schema rigor (versioned events, schema registry) matters more here than in any other pattern, since decoupled consumers can't negotiate format mismatches at runtime the way a tightly coupled supervisor call can.

### 3.4 State Machine Pattern

Model orchestration as explicit states.

```
Pending -> Researching -> Reviewing -> Approved -> Completed
```

- **Best for:** Compliance-heavy systems, regulated environments, long-running workflows.
- **Tools:** Temporal, LangGraph, Durable Functions.
- **Principal-level note:** Never let agents invent states dynamically. The set of valid states and valid transitions should be a fixed, versioned artifact reviewed the same way you'd review a database schema migration — because functionally, that's what it is.

### 3.5 Blackboard Architecture

Agents collaborate through shared memory.

```
Shared Context Store
    ↑    ↑    ↑
 Agent A B C D
```

- **Trade-off:** Flexible; context conflicts increase quickly.
- **Learn:** Memory versioning, conflict resolution, access controls.
- **Principal-level note:** This is the pattern most likely to degrade silently in production — conflicts don't throw errors, they just produce subtly wrong shared state that downstream agents trust. Treat every blackboard write as requiring the same rigor as a concurrent database write: explicit ownership, versioning, and conflict resolution rules decided in advance, not improvised at runtime.

### 3.6 Saga Pattern (Distributed Transaction Coordination)

*Not in the original draft — essential for any agent workflow that performs multiple side effects across systems and needs to handle partial failure.*

A saga coordinates a sequence of local actions across agents/services, with explicit compensating actions if a later step fails.

```
Step 1: Reserve inventory       -> Compensate: Release inventory
Step 2: Charge payment          -> Compensate: Refund payment
Step 3: Schedule shipment       -> Compensate: Cancel shipment
```

```json
{
  "saga_id": "saga_789",
  "steps": [
    { "step": "reserve_inventory", "status": "completed", "compensate_action": "release_inventory" },
    { "step": "charge_payment", "status": "failed", "compensate_action": "refund_payment" }
  ],
  "compensation_triggered": true
}
```

- **Best for:** Any multi-agent workflow with real-world side effects (payments, provisioning, notifications) that must not be left half-completed.
- **Trade-off:** Significant added complexity (you must design and test the compensating action for every step); but it's the only correct answer for distributed atomicity across agents that can't share a single transaction.
- **Principal-level note:** This is the single most commonly *missing* pattern in agent system designs that otherwise look sophisticated. If an interviewer asks "what happens if step 3 of your multi-agent workflow fails after steps 1 and 2 succeeded," and your answer doesn't involve a saga or equivalent compensation logic, that's the gap they'll keep probing.

### 3.7 Circuit Breaker Pattern (Applied to Agent/Tool Calls)

*Expanded from the brief mention in the original — this deserves its own schema and state model.*

Prevents a failing downstream agent or tool from being hammered with retries that waste cost and worsen the outage.

```
CLOSED (normal) -> [failure threshold exceeded] -> OPEN (fail fast, no calls)
       ^                                                ↓
       └── [success threshold met] <- HALF-OPEN <- [timeout elapsed]
```

```json
{
  "circuit_breaker": {
    "target": "crm_api_tool",
    "state": "open",
    "failure_count": 12,
    "failure_threshold": 10,
    "opened_at": "2026-06-21T12:00:00Z",
    "retry_after_seconds": 30,
    "fallback_action": "use_cached_response"
  }
}
```

- **Best for:** Any tool or sub-agent call that can fail under load — protects both cost (no runaway retries) and downstream system health.
- **Trade-off:** Adds state to track per tool/agent target; tuning thresholds requires production data, not guesswork.
- **Principal-level note:** Pair this explicitly with a defined fallback (cached response, alternate tool, human escalation) — a circuit breaker that just fails fast with no fallback only prevents one failure mode while introducing another (a hard stop in the workflow).

### 3.8 Bulkhead Pattern (Resource Isolation)

*New addition — directly relevant given your multi-tenant infrastructure background.*

Isolates resource pools per agent/tenant/workflow type so one consumer's failure or overload can't starve others.

```json
{
  "bulkhead_config": {
    "pool": "research_agent_pool",
    "max_concurrent_executions": 20,
    "max_queue_depth": 50,
    "isolated_from": ["executor_agent_pool", "review_agent_pool"]
  }
}
```

- **Best for:** Multi-tenant agent platforms, or any system where one workflow type misbehaving (e.g., a research agent stuck in a retrieval loop) shouldn't degrade unrelated workflows.
- **Trade-off:** Resource pools sized for isolation are individually less efficient than one shared pool — you're trading raw throughput for blast-radius containment.
- **Principal-level note:** This is the direct conceptual sibling to per-tenant rate limiting discussed in the multi-tenancy and zero-trust files — name that connection explicitly in an interview to show you see the same principle applied at different system layers.

### 3.9 Peer-to-Peer Agents

```
Agent A <-> Agent B <-> Agent C
```

- **Trade-off:** Flexibility vs. a debugging nightmare — avoid unless absolutely necessary.
- **Principal-level note:** Almost every production enterprise use case is better served by routing peer-to-peer-seeming interactions through a supervisor or event bus instead. If you find yourself defending a peer-to-peer design in an interview, be ready to explain specifically why centralized routing was rejected, not just that it was.

### 3.10 Dynamic Agent Creation

- **Best for:** Research environments, experimental systems — avoid in production unless you have a very specific, bounded justification.
- **Trade-off:** Adaptability vs. unbounded complexity.
- **Principal-level note:** Most production systems should never dynamically create agents. If asked about this capability, the strongest answer often acknowledges the theoretical appeal while explaining why you'd resist it without an extremely narrow, well-governed exception case.

---

## 4. Inter-Agent Schema Catalog

The biggest mistake teams make is allowing agents to exchange arbitrary text. Expert systems use structured schemas with explicit inputs/outputs, versioned contracts, provenance metadata, state transitions, correlation IDs, and audit trails. Treat agents as distributed microservices with probabilistic behavior.

### 4.1 Core Metadata Envelope (Mandatory on Every Message)

```json
{
  "message_id": "msg_123",
  "correlation_id": "workflow_456",
  "causation_id": "msg_122",
  "parent_task_id": "task_789",
  "sender_agent": "research_agent",
  "receiver_agent": "review_agent",
  "workflow_id": "wf_001",
  "workflow_version": "2.1",
  "timestamp": "2026-06-21T12:00:00Z",
  "schema_version": "1.0",
  "priority": "high",
  "ttl_seconds": 1800,
  "trace_id": "otel_trace_abc",
  "span_id": "otel_span_xyz"
}
```

Enables: end-to-end tracing, distributed debugging, replay capabilities, failure recovery, dependency analysis.

### 4.2 Request-Response Pattern

```json
{
  "request": { "task": "summarize_document", "parameters": {} },
  "response": { "status": "success", "result": {} }
}
```
Use for synchronous, single-responsibility, low-latency tasks. Easy to debug; limited scalability.

### 4.3 Command Pattern

```json
{ "command": { "action": "create_ticket", "arguments": {} } }
```
Imperative, strong ownership, clear accountability. Best for operational workflows, tool execution, automation pipelines.

### 4.4 Event Pattern

```json
{ "event": { "type": "invoice_approved", "payload": {} } }
```
Decoupled, asynchronous, reactive. Best for enterprise automation, event-driven architectures, long-running workflows.

### 4.5 Task Contract Pattern

```json
{
  "task": {
    "task_id": "task_001",
    "objective": "review_security_findings",
    "inputs": {},
    "constraints": {},
    "expected_outputs": {},
    "deadline": "2026-06-21T14:00:00Z"
  }
}
```
Benefits: clear expectations, better evaluations, easier delegation.

### 4.6 Planner-Executor Schema

Planner output:
```json
{
  "plan_id": "plan_001",
  "steps": [
    { "step_id": 1, "agent": "research_agent", "objective": "collect_sources" },
    { "step_id": 2, "agent": "analysis_agent", "objective": "synthesize_findings" }
  ]
}
```
Executor input:
```json
{ "plan_id": "plan_001", "step_id": 1 }
```
Benefits: reproducibility, replayability, human approval checkpoints.

### 4.7 Blackboard Pattern

```json
{
  "memory_key": "market_analysis",
  "owner": "research_agent",
  "version": 7,
  "content": {},
  "last_updated": "",
  "confidence": 0.91
}
```
Must add ownership, versioning, access control, expiration — without these, shared memory becomes chaos.

### 4.8 State Machine Pattern

```json
{
  "entity_id": "case_001",
  "current_state": "security_review",
  "previous_state": "analysis",
  "allowed_next_states": ["approved", "rejected"]
}
```
Benefits: compliance, recovery, auditing. Never let agents invent states dynamically.

### 4.9 Observation-Thought-Action Pattern (Production Variant)

```json
{
  "observation": {},
  "decision_summary": {},
  "action": {}
}
```
**Avoid storing raw chain-of-thought.** Persist inputs, decision summaries, and outputs — not internal reasoning. This matters both for cost (raw reasoning traces are token-expensive to store and retrieve) and for governance (a decision summary is auditable; an unstructured reasoning dump usually isn't).

### 4.10 Human-in-the-Loop Pattern

```json
{
  "approval": {
    "required": true,
    "approver_role": "security_manager",
    "reason": "high_risk_action",
    "expires_at": ""
  }
}
```
Track who approved, when approved, and what changed after approval.

### 4.11 Evaluation Pattern

```json
{
  "evaluation": {
    "accuracy": 0.93,
    "confidence": 0.88,
    "cost_usd": 0.07,
    "latency_ms": 2140,
    "success": true
  }
}
```
Every agent response should be measurable. No metrics means no improvement.

### 4.12 Provenance Pattern

```json
{
  "provenance": [
    {
      "source_type": "document",
      "source_id": "doc_123",
      "chunk_id": "chunk_45",
      "retrieved_at": "",
      "confidence": 0.94
    }
  ]
}
```
Essential for RAG, compliance, and explainability. You should always be able to answer: where did this answer come from? Which agent modified it? Which tool generated it?

### 4.13 Tool Invocation Pattern

```json
{
  "tool_call": { "tool_name": "crm_api", "parameters": {}, "permissions": ["read_customer"] }
}
```
```json
{
  "tool_result": { "status": "success", "execution_time_ms": 120, "output": {} }
}
```
Always log inputs, outputs, errors, and permissions actually used.

### 4.14 Error Pattern

```json
{
  "error": {
    "code": "TOOL_TIMEOUT",
    "message": "CRM API timeout",
    "retryable": true,
    "retry_count": 2,
    "fallback_agent": "backup_agent"
  }
}
```
Benefits: automatic recovery, better analytics, reduced MTTR.

### 4.15 Confidence Pattern

```json
{
  "confidence": {
    "overall": 0.82,
    "dimensions": { "factual": 0.91, "completeness": 0.77, "relevance": 0.85 }
  }
}
```
Use confidence for routing, escalation, and human-approval thresholds — not just as a reported metric.

### 4.16 Capability Registry Pattern

```json
{
  "agent_id": "security_agent",
  "capabilities": ["threat_analysis", "policy_validation"],
  "accepted_schemas": ["task.v2"],
  "output_schemas": ["finding.v3"]
}
```
Treat agents like discoverable services. Benefits: dynamic routing, plug-and-play agents, multi-team scalability.

### 4.17 Context Window Pattern

```json
{
  "context": {
    "documents": [],
    "memory_refs": [],
    "tools": [],
    "token_budget": 24000,
    "compression_strategy": "summary"
  }
}
```
Critical for cost control, debugging, and hallucination reduction — track exactly what context each agent received, not just what it produced.

### 4.18 Policy Enforcement Pattern

```json
{
  "policies": {
    "data_classification": "confidential",
    "allowed_tools": ["search"],
    "restricted_actions": ["delete_records"]
  }
}
```
**Agents should not decide their own permissions.** Policy enforcement is separate from agent logic — this is the architectural embodiment of "ask should, not can" from Section 2.

### 4.19 Recommended Unified Workflow Schema

```
Envelope
|-- Context
|-- Task
|-- Constraints
|-- Policies
|-- Tool Calls
|-- Outputs
|-- Confidence
|-- Provenance
|-- Evaluation
`-- Audit Metadata
```

### 4.20 Minimum Fields for Enterprise Traceability

If you only implement ten fields, make them these — they solve most debugging problems:

1. `message_id`
2. `correlation_id`
3. `causation_id`
4. `workflow_id`
5. `agent_id`
6. `task_id`
7. `schema_version`
8. `timestamp`
9. `trace_id`
10. `provenance`

### 4.21 Open Standards Worth Studying

OpenTelemetry, CloudEvents, OpenAPI Specification, JSON Schema, Apache Avro, AsyncAPI, OpenLineage, Temporal, LangGraph

**The governing test:** every agent decision should be explainable, reproducible, measurable, and auditable. If you cannot answer *what happened, why it happened, who made the decision, which data influenced it,* and *can we replay it* — you don't have orchestration yet. You have conversations between black boxes.

---

## 5. Complexity & Dynamics Reduction Strategies

Reducing dynamics means reducing non-determinism, emergent behavior, uncontrolled autonomy, context variability, tool unpredictability, workflow branching, and hidden state changes.

**Governing principle:** every dynamic behavior should exist because you explicitly designed it — not because the model invented it. The goal is not maximum autonomy. The goal is predictable, auditable, bounded autonomy.

### 5.1 The Agent Stability Hierarchy

Order patterns from most predictable to least predictable. Start at the top; move downward only when necessary. Most enterprise use cases never need peer-to-peer agents.

```
Deterministic Workflow
    ↓
State Machine
    ↓
Planner -> Executor
    ↓
Supervisor Pattern
    ↓
Event-Driven Agents
    ↓
Blackboard Pattern
    ↓
Peer-to-Peer Agents
    ↓
Dynamic Agent Creation
```

### 5.2 Universal Rule: Reduce Degrees of Freedom

Every dynamic dimension increases complexity. Control: number of agents, tools, memory stores, handoffs, model choices, possible states, retries, and context sources.

```
Complexity ≈ Agents × Tools × States × Handoffs × Context Sources
```

Reducing any single factor dramatically improves reliability — this is multiplicative, not additive, so even one factor cut in half meaningfully reduces total system complexity.

### 5.3 Strategy Catalog

**Strategy 1 — Replace conversations with contracts.** Avoid `Agent A → free text → Agent B`. Prefer an explicit task contract (Section 4.5). Benefits: reduced ambiguity, better debugging, easier replay, lower hallucination rates.

**Strategy 2 — Use finite state machines.** Never allow arbitrary transitions. Enforce `allowed_next_states` explicitly (Section 4.8). Benefits: deterministic behavior, easier compliance, better recovery.

**Strategy 3 — Centralize routing.** Avoid peer-to-peer communication; route through a supervisor. Benefits: single audit point, reduced coordination failures, simpler observability. Trade-off: supervisor bottlenecks (accept this deliberately — see 3.1).

**Strategy 4 — Limit agent specialization.** Avoid 25 narrow agents; prefer 3-7 well-defined agents (Planner, Domain Expert, Tool Executor, Reviewer, Compliance Agent is a typical enterprise set). Benefits: lower maintenance cost, fewer handoffs, simpler testing.

**Strategy 5 — Minimize context variability.** Avoid dynamic prompts, unlimited memory, uncontrolled retrieval. Prefer a fixed structure used every time:
```
System Instructions
Task Context
Retrieved Documents (Top 5)
Memory Summary
User Input
```
Always use the same order — inconsistent context ordering produces inconsistent model behavior even with identical content.

**Strategy 6 — Control tool access explicitly per agent.**
```json
{
  "research_agent": ["search"],
  "executor_agent": ["crm_write"],
  "reviewer_agent": []
}
```
Benefits: reduced attack surface, lower failure rates, better governance.

**Strategy 7 — Standardize memory into three clear types.**

| Memory | Purpose |
|---|---|
| Working | Current workflow |
| Long-term | Historical knowledge |
| Audit | Immutable records |

Avoid sprawl across multiple ad hoc stores (Vector DB A, Redis B, SQL C, File D) with no clear ownership boundary.

**Strategy 8 — Fix model selection via policy, not agent discretion.**

| Task | Model |
|---|---|
| Classification | Small model |
| Planning | Large model |
| Retrieval | Embedding model |

Benefits: predictable cost, consistent outputs, easier benchmarking.

**Strategy 9 — Bound planning depth.** Avoid recursive planning (`Planner → Planner → Planner`). Set explicit limits:
```json
{ "max_depth": 3, "max_iterations": 5 }
```
Benefits: prevents loops, controls cost, reduces latency.

**Strategy 10 — Use confidence thresholds to gate action.**
```
>= 0.9   -> Execute
0.7-0.9  -> Review
< 0.7    -> Human approval
```
Benefits: reduced errors, safer automation.

**Strategy 11 — Prefer event sourcing.** Store all decisions as immutable events (`TaskCreated`, `TaskAssigned`, `ToolCalled`, `TaskCompleted`). Never overwrite state; derive current state from the event log. Benefits: replayability, debugging, compliance. Trade-off: more storage, higher implementation effort.

**Strategy 12 — Design for failure as the default assumption.** Assume tools fail, models hallucinate, APIs time out, memory corrupts. Implement timeouts, retries, circuit breakers (3.7), dead-letter queues, fallback models — and for multi-step side-effecting workflows, sagas (3.6).

---

## 6. Pattern-Specific Recommendations

| Pattern | Keep | Avoid |
|---|---|---|
| **Single Agent** | One model, few tools, fixed prompts | Memory sprawl, dynamic tool discovery |
| **Planner-Executor** | Deterministic plans, maximum steps, fixed executors | Executor replanning |
| **Supervisor** | Central routing, fixed specialists | Agent-to-agent messaging |
| **Event-Driven** | Immutable events, schema registry | Unversioned events |
| **Blackboard** | Ownership rules, version control | Shared write access |
| **Peer-to-Peer** | Strict communication rules if used at all | Use unless absolutely necessary |
| **Dynamic Agent Creation** | — | Avoid unless research/experimental context |

**Trade-off summary for the two highest-risk patterns:**

| Pattern | Benefit | Cost |
|---|---|---|
| Peer-to-Peer | Flexibility | Debugging nightmare |
| Dynamic Agent Creation | Adaptability | Unbounded complexity |

Most production systems should never dynamically create agents.

---

## 7. Economics, Observability & Security

### 7.1 The Economics of Agents

An expert orchestrator thinks in tokens, latency, and ROI — track cost per task, cost per user, cost per workflow, tokens per decision, latency per step, and success rate.

**Optimization techniques:** context compression, semantic caching, model routing, tool selection policies, adaptive retrieval, small-model-first strategies.

**The governing heuristic:** a cheaper workflow often beats a smarter workflow. In an interview, this is the answer to "how do you decide when to use a smaller model" — frame it as a deliberate economics decision, not a capability compromise.

### 7.2 Agent Observability

You cannot orchestrate what you cannot measure. Track: task completion rate, tool success rate, hallucination frequency, human override rate, memory hit rate, latency distribution, token consumption, and agent interaction graphs.

Build dashboards for: workflow traces, decision paths, failure analysis.

**Study:** OpenTelemetry, Langfuse, Arize, Helicone, Weights & Biases. *(See the companion `Observability_Evaluation_Architecture.md` for the full tracing schema and eval framework treatment.)*

### 7.3 Security as the Expert/Enthusiast Boundary

Design explicitly for: agent identity, agent-to-agent authentication, least privilege, tool authorization, secret isolation, memory access controls, prompt injection defense, data classification, audit trails.

The governing question, repeated from Section 2 because it's the single highest-signal framing in this entire document: **"Should this agent be allowed to do this action?"** — not "can this agent do this action?" *(See the companion `IAM_ZeroTrust_Agent_Architecture.md` for the full identity and policy enforcement treatment.)*

### 7.4 Evaluation Frameworks

Never trust demos. Create evaluation datasets and measure: goal completion %, accuracy %, cost per success, human intervention rate, mean time to recovery.

Run regression tests, adversarial tests, prompt injection tests, and chaos engineering experiments. **Break your agents deliberately** — this is the practice that separates a system you've validated from a system you merely hope works.

---

## 8. The Enterprise Agent Stack

For maximum stability, structure the full stack like this:

```
User
 ↓
API Gateway
 ↓
Policy Engine
 ↓
Workflow Engine
 ↓
Supervisor
 ↓
Specialist Agents
 ↓
Tools
```

**Cross-cutting concerns** — authentication, authorization, observability, evaluation, audit logging — live outside the agents, not embedded inside them. This is the single most important structural decision in the whole stack: cross-cutting concerns implemented per-agent will drift out of sync with each other; implemented as shared infrastructure, they stay consistent by construction.

---

## 9. Decision Framework & Closing Heuristics

### 9.1 Questions to Ask Before Adding Autonomy

1. Can a workflow solve this?
2. Can a state machine solve this?
3. Can rules solve this?
4. Do I really need another agent?
5. Can I remove a tool?
6. Can I eliminate a handoff?

**The best orchestration systems remove complexity instead of adding intelligence.**

### 9.2 Questions Expert Orchestrators Ask Constantly

- What happens when an agent fails?
- Who owns state?
- Can this workflow recover automatically?
- How is success measured?
- What is the cost of each decision?
- What are the security boundaries?
- Where is human approval required?
- Can smaller models do this task?
- What is the blast radius?

### 9.3 The Closing Heuristic

Aim for:
- Maximum determinism
- Minimum autonomy
- Sufficient intelligence

**Not** maximum intelligence. Enterprise success depends more on reliability, explainability, recoverability, and governance than on agent creativity.

When you consistently think this way, you stop building AI features and start designing autonomous systems. That mindset is what distinguishes expert-level agent orchestration architects from everyone else building on the same frameworks.

---

## Companion Documents

This file is part of a matched set covering Principal AI Engineer / FDE architecture depth:

- `RAG_Architecture_Deep_Dive.md` — retrieval patterns, chunking schemas, hybrid search contracts
- `Model_Serving_Architecture_Deep_Dive.md` — serving patterns, batching schemas, multi-tenancy
- `Fine_Tuning_Workflow_Architecture.md` — QLoRA/Axolotl pipeline schemas, evaluation gates
- `IAM_ZeroTrust_Agent_Architecture.md` — agent identity, credential lifecycle, policy enforcement
- `AI_Governance_Compliance_Schemas.md` — EU AI Act/DORA/NIS2 evidence and audit schemas
- `Observability_Evaluation_Architecture.md` — tracing schemas, eval frameworks, cost dashboards
