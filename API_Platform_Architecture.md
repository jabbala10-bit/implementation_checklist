# API & Platform Architecture — REST/GraphQL/gRPC Tradeoffs, Gateway Patterns & Versioning Strategy

**Principal Engineer / AI Architect / FDE Reference · Gunasekar Jabbala**

> Every backend system in this series eventually needs to expose an interface to something else - a client application, another internal service, or a third-party integration at a client site. For an FDE specifically, this is the most frequently hands-on category: integrating into a client's existing, often messy API surface is more common day-to-day work than designing a clean new one from scratch.

---

## Table of Contents

1. [The API Architecture Maturity Model](#1-the-api-architecture-maturity-model)
2. [REST, GraphQL, and gRPC - A Real Tradeoff Analysis](#2-rest-graphql-and-grpc--a-real-tradeoff-analysis)
3. [API Gateway Patterns](#3-api-gateway-patterns)
4. [Versioning & Backward Compatibility Strategy](#4-versioning--backward-compatibility-strategy)
5. [The FDE-Specific Skill: Integrating Into Messy Existing Systems](#5-the-fde-specific-skill-integrating-into-messy-existing-systems)
6. [Complexity Reduction for API Architecture Specifically](#6-complexity-reduction-for-api-architecture-specifically)
7. [Decision Framework](#7-decision-framework)

---

## 1. The API Architecture Maturity Model

| Level | Name | Capabilities |
|---|---|---|
| **1** | Ad Hoc Endpoints | APIs built per-need, inconsistent conventions, no formal versioning |
| **2** | Consistent Style | A single chosen API style (REST/GraphQL/gRPC) applied consistently, basic versioning |
| **3** | Governed Platform | API gateway enforcing policy centrally, formal versioning and deprecation process, contract-first design |
| **4** | Self-Service API Platform | Internal/external developer portal, automated compatibility testing, API products treated as managed assets with their own lifecycle and ownership |

The FDE-specific reality worth naming directly: you'll rarely get to operate at a clean Level 3-4 when integrating into a client's existing systems — you'll often be working with their Level 1-2 reality while trying to build something Level 3-quality on top of it. The skill isn't insisting the client fix their API maturity first; it's architecting your integration layer to be robust despite their maturity level (Section 5).

---

## 2. REST, GraphQL, and gRPC - A Real Tradeoff Analysis

### 2.1 The Tradeoffs That Actually Matter, Not the Generic Comparison Chart

**REST:** Simple, cacheable (via standard HTTP semantics), universally understood, but prone to over-fetching (getting more data than needed) or under-fetching (needing multiple round trips to assemble what a client actually needs) when the resource model doesn't match the client's actual access pattern.

**GraphQL:** Solves over/under-fetching directly — clients specify exactly the fields they need across what would otherwise be multiple REST calls. The real cost: query complexity becomes the client's responsibility to express correctly, and naive GraphQL implementations are vulnerable to clients constructing deeply nested queries that cause server-side performance problems disproportionate to a single request's apparent simplicity (the N+1 query problem, and deliberately malicious deeply-nested query attacks).

**gRPC:** Binary protocol (Protocol Buffers), strongly typed contracts, genuinely efficient for service-to-service communication, with native streaming support — but not natively browser-friendly (requires a proxy layer like grpc-web for browser clients) and less human-debuggable than REST/JSON without tooling.

```json
{
  "api_style_decision": {
    "use_case": "internal_agent_to_tool_communication",
    "chosen_style": "grpc",
    "rationale": "high call volume, strongly-typed contracts matter for schema enforcement, streaming support fits tool-call and response patterns",
    "rejected_alternatives": {
      "rest": "over/under-fetching less relevant for one well-defined call per tool, but lacks strong typing without additional schema validation layers",
      "graphql": "query flexibility is unnecessary overhead for well-defined, fixed-shape tool calls"
    }
  }
}
```

**Principal-level note:** the strongest answer to "which should we use" names the specific property of the use case driving the choice — call volume and type-safety needs for internal service mesh communication (favoring gRPC), heterogeneous client needs with varying data requirements (favoring GraphQL), or broad external consumability and cacheability (favoring REST) — rather than a general preference for one style across all use cases.

### 2.2 The Hybrid Reality of Most Real Platforms

**Principal-level note:** most mature platforms use more than one style simultaneously and deliberately — gRPC for internal service-to-service calls where performance and type safety matter most, REST or GraphQL for external-facing APIs where broad client compatibility matters more than peak efficiency. Presenting this as a deliberate, use-case-driven hybrid strategy is a stronger answer than defending a single style as universally correct.

---

## 3. API Gateway Patterns

### 3.1 What an API Gateway Actually Centralizes

```json
{
  "gateway_policy": {
    "route": "/v2/fraud-detection/score",
    "backend_service": "fraud-detection-api",
    "rate_limit": { "requests_per_minute": 1000, "per": "api_key" },
    "auth_requirement": "oauth2_bearer_token",
    "request_transformation": "inject_internal_trace_id_header",
    "response_caching": { "enabled": false, "reason": "real_time_fraud_score_must_not_be_cached" }
  }
}
```

**Principal-level note:** this is the API-layer concrete implementation of the IAM and Zero-Trust document's centralized Policy Decision Point pattern — auth, rate limiting, and policy enforcement happen once, centrally, at the gateway, rather than being reimplemented inconsistently inside every individual backend service. The explicit response_caching false with a stated reason is itself worth noting as a deliberate decision artifact, not an oversight — caching a fraud score response would be a serious correctness bug, and documenting why it's disabled prevents someone later optimizing by adding caching back without understanding the constraint.

### 3.2 The Backend-for-Frontend (BFF) Pattern

A specific gateway variant: rather than one generic gateway serving all client types identically, a BFF is a dedicated gateway layer tailored to a specific client (mobile app, web app, partner integration), aggregating and shaping backend responses specifically for that client's needs.

**Principal-level note:** this matters specifically for AI product surfaces — a chat interface client and a programmatic API client consuming the same underlying agent orchestration platform often need genuinely different response shapes (streaming token-by-token for the chat UI, a single structured JSON response for the API client) — a BFF per client type avoids forcing one generic response format to awkwardly serve both.

---

## 4. Versioning & Backward Compatibility Strategy

### 4.1 The Core Principle - Additive Changes vs. Breaking Changes

Adding a new optional field to a response is additive and safe; removing a field, renaming a field, or changing a field's type or meaning is breaking. This distinction is the same backward-compatibility logic as the Data Engineering document's schema registry compatibility checks, applied to API contracts instead of streaming schemas.

```json
{
  "api_version_policy": {
    "versioning_strategy": "uri_path_versioning",
    "current_version": "v2",
    "deprecated_versions": [{ "version": "v1", "sunset_date": "2026-12-31", "active_consumers": 14 }],
    "breaking_change_policy": "new_major_version_required, minimum_6_month_deprecation_window"
  }
}
```

**Principal-level note:** active_consumers being tracked explicitly is what makes a sunset decision defensible and safe — deprecating an API version without knowing how many consumers still depend on it is the API-layer version of the same blast-radius-blindness problem the Data Engineering document's data contracts solve for streaming schemas.

### 4.2 URI Versioning vs. Header Versioning vs. No Versioning (Evolve-Only)

URI versioning (a version segment in the path) is the most visible and simplest to reason about, but proliferates URL surface area over time. Header versioning (a version specified in a request header) keeps URLs stable but is less discoverable and debuggable. Evolve-only (never break compatibility, only add fields, no version number at all) is the GraphQL-favored approach and avoids version proliferation entirely, but requires very strict discipline that no field is ever removed or repurposed.

**Principal-level note:** the right choice correlates with how much control you have over consumers — for a tightly controlled internal API where you can coordinate consumer upgrades, evolve-only with strict additive discipline is often cleanest; for a broadly externally-consumed API where you can't control or even know all consumers, explicit URI versioning with a long deprecation window is the safer, more conservative choice.

---

## 5. The FDE-Specific Skill: Integrating Into Messy Existing Systems

### 5.1 The Adapter/Anti-Corruption Layer Pattern

When integrating with a client's legacy or inconsistent API, the correct architectural response is building an explicit adapter layer that normalizes the messy external interface into a clean internal contract — rather than letting the external system's inconsistencies leak into and pollute your own internal architecture.

```json
{
  "anti_corruption_layer": {
    "external_system": "legacy_client_crm_api",
    "external_quirks_absorbed": [
      "inconsistent pagination, offset-based on one endpoint, cursor-based on another",
      "undocumented rate limit discovered empirically",
      "occasional duplicate webhook delivery with no idempotency key provided by the external system"
    ],
    "internal_contract_exposed": "clean, consistent, paginated, deduplicated interface matching internal standards",
    "absorbed_complexity_documented_for_client": true
  }
}
```

**Principal-level note:** absorbed_complexity_documented_for_client being true reflects a specific FDE value-add — not just solving the immediate integration problem, but documenting the external system's actual, often undocumented, behavior for the client's own team, leaving them better positioned than before the engagement, which is a recurring theme worth carrying into how you discuss any integration work.

### 5.2 Defensive Integration - Assuming the External System Will Misbehave

Given a client's external API, defensive practices worth building in by default: explicit timeouts (never trust a default), retries with backoff for transient failures, circuit breakers for sustained failures, and idempotency keys generated on your side when the external system doesn't provide its own deduplication mechanism — directly addressing the duplicate webhook delivery scenario above.

---

## 6. Complexity Reduction for API Architecture Specifically

| Degree of Freedom | Reduction Strategy |
|---|---|
| Number of API styles in active use | A deliberate, small set (e.g., gRPC internal, REST external) with documented rationale per style, not ad hoc style choice per team |
| Versioning surface area | One consistent versioning strategy platform-wide, not different approaches per service |
| Gateway policy duplication | Centralized gateway policy for cross-cutting concerns (auth, rate limiting), not reimplemented per-service |
| Integration-layer fragility | One adapter/anti-corruption layer per messy external system, isolating its quirks, rather than letting external inconsistency leak into multiple internal consumers independently |

---

## 7. Decision Framework

1. Does this specific use case's access pattern favor REST's simplicity, GraphQL's flexible field selection, or gRPC's performance and type safety, derived from actual call volume, client diversity, and latency needs, not general preference?
2. Is there a documented, enforced versioning and deprecation policy with known consumer counts per version, or would deprecating an old API version be a surprise to consumers still depending on it?
3. For a client integration, is there an explicit adapter layer absorbing the external system's quirks, or is that messiness leaking directly into your internal architecture?
4. Are cross-cutting concerns (auth, rate limiting, policy) centralized at a gateway layer, or reimplemented inconsistently inside individual backend services?
5. Have you built defensive timeout, retry, and idempotency handling for any external integration, assuming it will misbehave, rather than assuming it will always behave as documented?

**The governing test:** an API surface, whether one you're designing fresh or one you're integrating into at a client site, should have its compatibility guarantees and consumer blast radius known and tracked, the same auditability principle running through every other document in this series, applied here to the contracts between systems rather than within a single system.

---

## Companion Documents

Part of the Principal Engineer / AI Architect / FDE core engineering series:

- `Agent_Orchestration_Architecture.md` — the schema enforcement and tool invocation patterns that map directly onto gRPC contract design
- `IAM_ZeroTrust_Agent_Architecture.md` — the centralized policy decision point pattern implemented concretely at the gateway layer
- `Data_Engineering_Streaming_Architecture.md` — the schema registry compatibility checking principle applied here to API versioning
- `Cloud_Native_Kubernetes_Architecture.md` — service mesh traffic management as a complementary layer to gateway-level API policy
- `Principal_AI_FDE_Coding_Challenges.md` — the FDE live-build pagination inconsistency scenario this document's anti-corruption layer pattern directly addresses
