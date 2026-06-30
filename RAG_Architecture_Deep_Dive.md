# RAG Architecture Deep Dive — Retrieval Patterns, Schemas & Production Failure Modes

**Principal AI Engineer / FDE Reference · Gunasekar Jabbala**

> RAG is a retrieval-quality problem wearing a generation-quality costume. Most teams debug a "bad answer" by tuning the prompt; experts debug it by checking what was actually retrieved first. Treat every retrieval as a contract — explicit inputs, explicit outputs, explicit provenance — the same discipline this series applies to inter-agent messages.

---

## Table of Contents

1. [The RAG Maturity Model](#1-the-rag-maturity-model)
2. [Retrieval Architectural Patterns](#2-retrieval-architectural-patterns)
3. [Chunking & Indexing Schemas](#3-chunking--indexing-schemas)
4. [Hybrid Search & Fusion Contracts](#4-hybrid-search--fusion-contracts)
5. [Production Failure Modes & Mitigations](#5-production-failure-modes--mitigations)
6. [Complexity Reduction for RAG Specifically](#6-complexity-reduction-for-rag-specifically)
7. [Decision Framework](#7-decision-framework)

---

## 1. The RAG Maturity Model

| Level | Name | Capabilities |
|---|---|---|
| **1** | Naive RAG | Fixed-size chunking, single embedding model, top-k cosine similarity, no evaluation |
| **2** | Tuned RAG | Semantic chunking, hybrid search (dense + sparse), re-ranking, basic evaluation set |
| **3** | Self-Correcting RAG | Retrieval grading (CRAG), query rewriting, fallback sources, confidence-gated generation |
| **4** | Governed Production RAG | Multi-tenant isolation, full provenance audit trail, continuous evaluation, freshness/staleness SLAs, access-control-aware retrieval |

Most teams ship at Level 1-2 and discover the gap to Level 3-4 only after a hallucination incident. The gap is almost never "the model isn't good enough" — it's missing grading, provenance, and evaluation infrastructure.

---

## 2. Retrieval Architectural Patterns

### 2.1 Naive Top-K Retrieval

```
Query -> Embed -> Cosine Similarity -> Top K Chunks -> Stuff into Prompt
```

- **Best for:** Prototypes, low-stakes internal tools, proof-of-concept demos.
- **Trade-off:** Simple, fast to build; no resilience to poor retrieval — a bad top-k result silently produces a bad answer with no signal anything went wrong.
- **Principal-level note:** This is the right starting point for a take-home or live-build exercise specifically because it's fast — but say out loud that you know it's Level 1, and name what you'd add next. Volunteering the gap is the signal, not avoiding building the simple version.

### 2.2 Corrective RAG (CRAG)

```
Query -> Retrieve -> Grade Relevance -> [sufficient? Generate] / [insufficient? Fallback (web search / broader retrieval)]
```

```json
{
  "retrieval_grade": {
    "query_id": "q_001",
    "documents_graded": 5,
    "relevant_count": 2,
    "relevance_threshold": 0.6,
    "decision": "fallback_triggered",
    "fallback_source": "web_search"
  }
}
```

- **Best for:** Production systems where retrieval quality varies by query type and silent failure is costly.
- **Trade-off:** Added latency and cost from the grading step (often a smaller, cheaper model than the main generator) — but it converts an invisible failure mode into a visible, handleable one.
- **Principal-level note:** The grading step should use a cheaper model than the generation step. Grading every document with the same large model you generate with is a common and avoidable cost mistake.

### 2.3 Self-RAG

```
Query -> [Reflect: is retrieval needed?] -> [No: answer directly] / [Yes: Retrieve -> Generate -> Reflect: is this grounded?] -> [No: retry] / [Yes: return]
```

- **Best for:** Mixed query loads where some queries need no retrieval at all (general knowledge, conversational) and forcing retrieval on every query wastes latency and money.
- **Trade-off:** Requires the model to make a well-calibrated retrieval-necessity judgment — a poorly calibrated model either retrieves unnecessarily (wasted cost) or skips retrieval when it shouldn't (hallucination).
- **Principal-level note:** This pattern's reflection step needs its own evaluation, separate from end-to-end answer quality — you specifically need to measure whether the retrieval-necessity decision itself is well-calibrated, since that's a distinct failure mode from generation quality.

### 2.4 Agentic Multi-Hop Retrieval

```
Query -> Decompose into sub-questions -> Retrieve per sub-question -> Synthesize across results -> [More needed? Loop] / [Sufficient? Answer]
```

```json
{
  "multi_hop_state": {
    "original_query": "Compare the EU AI Act and DORA incident reporting timelines",
    "sub_queries": [
      { "id": "sq_1", "text": "EU AI Act serious incident reporting timeline", "status": "answered" },
      { "id": "sq_2", "text": "DORA incident reporting timeline", "status": "answered" }
    ],
    "synthesis_ready": true,
    "hop_count": 2,
    "max_hops": 4
  }
}
```

- **Best for:** Queries genuinely requiring information from multiple, non-adjacent sources that a single retrieval pass can't surface together.
- **Trade-off:** Latency scales with hop count; needs a hard `max_hops` cap (same principle as bounding planning depth in the orchestration file) or it risks unbounded cost on ambiguous queries.
- **Principal-level note:** Tie this explicitly back to the orchestration document's Strategy 9 (bound planning depth) when discussing it — multi-hop retrieval is the planner-executor pattern applied specifically to retrieval, and the same complexity-reduction discipline applies.

### 2.5 Conversational / Contextual RAG

```
Query + Conversation History -> Rewrite query with resolved context -> Retrieve -> Generate (with conversation history in context)
```

- **Best for:** Multi-turn assistants where a follow-up question ("what about for automotive?") only makes sense given prior turns.
- **Trade-off:** Query rewriting adds a step and a failure point — a bad rewrite produces a confidently wrong retrieval with no obvious symptom.
- **Principal-level note:** Log both the original query and the rewritten query in the provenance record (Section 3, orchestration file). When debugging a bad answer in this pattern, the rewrite is the first thing to check, and you can't check it if it wasn't logged.

---

## 3. Chunking & Indexing Schemas

### 3.1 Chunk Metadata Envelope (Mandatory)

```json
{
  "chunk_id": "chunk_4521",
  "document_id": "doc_998",
  "document_version": "3",
  "chunk_index": 7,
  "chunk_strategy": "semantic",
  "char_start": 1840,
  "char_end": 2210,
  "embedding_model": "text-embedding-3-large",
  "embedding_model_version": "2026-03",
  "ingested_at": "2026-05-10T09:00:00Z",
  "source_type": "pdf",
  "access_tags": ["tenant_acme", "classification_internal"],
  "freshness_expires_at": "2026-12-10T09:00:00Z"
}
```

- **Why each field matters:** `document_version` lets you detect and purge stale chunks when a source document updates. `embedding_model_version` matters because re-embedding with a new model invalidates direct comparison against old vectors — mixing embedding versions in one index silently degrades retrieval quality. `access_tags` is what makes retrieval access-control-aware rather than a separate bolt-on filter. `freshness_expires_at` lets you build automated staleness-based fallback (Section 5.3).

### 3.2 Semantic Chunking Contract

```json
{
  "chunking_job": {
    "document_id": "doc_998",
    "strategy": "semantic",
    "similarity_threshold": 0.5,
    "max_chunk_tokens": 512,
    "min_chunk_tokens": 64,
    "overlap_tokens": 50,
    "boundary_signals": ["heading", "paragraph_break", "topic_shift_embedding"]
  }
}
```

- **Principal-level note:** Cheap heuristic boundary signals (headings, paragraph breaks) should run *before* the expensive embedding-based topic-shift detection — only fall back to embedding comparison on ambiguous spans. Embedding every sentence to find boundaries is the single most common unnecessary cost in chunking pipelines.

### 3.3 Structured Content Chunk (Tables, Code)

```json
{
  "chunk_id": "chunk_4522",
  "content_type": "table",
  "serialization_format": "markdown_table",
  "preserves_structure": true,
  "raw_content_ref": "doc_998_table_3",
  "fallback_text_summary": "Pricing table comparing tiers A, B, C across three regions"
}
```

- **Principal-level note:** Naive text chunking destroys row-column relationships in tables. Tables and code need a distinct chunk type with their own serialization logic — flag this explicitly as a design decision when discussing RAG for enterprise document sets, since interviewers specifically probe whether candidates have hit this failure mode in practice.

### 3.4 Index Versioning Contract

```json
{
  "index_version": {
    "version_id": "idx_v12",
    "embedding_model_version": "2026-03",
    "chunking_strategy_version": "2.1",
    "document_count": 48213,
    "built_at": "2026-06-01T00:00:00Z",
    "status": "active",
    "previous_version": "idx_v11",
    "rollback_supported": true
  }
}
```

- **Principal-level note:** Treat the index itself as a versioned, rollback-able artifact — the same discipline as model deployment versioning. A retrieval quality regression after a re-index should be as fast to roll back as a bad model deployment, which requires keeping the previous index version warm, not just deleting it on rebuild.
---

## 4. Hybrid Search & Fusion Contracts

### 4.1 Hybrid Retrieval Request

```json
{
  "hybrid_query": {
    "query_id": "q_4521",
    "raw_query": "EU AI Act incident reporting deadline",
    "dense_search": { "embedding_model": "text-embedding-3-large", "top_k": 20 },
    "sparse_search": { "algorithm": "bm25", "top_k": 20 },
    "metadata_filter": { "access_tags": ["tenant_acme"], "document_type": "regulation" }
  }
}
```

- **Principal-level note:** Metadata filtering should apply to *both* the dense and sparse search legs before fusion, not just to the final combined result — filtering after fusion wastes retrieval budget on candidates that were never eligible to begin with, and can crowd out genuinely relevant filtered-in results if `top_k` is fixed.

### 4.2 Reciprocal Rank Fusion (RRF) Contract

```json
{
  "fusion_result": {
    "query_id": "q_4521",
    "method": "reciprocal_rank_fusion",
    "k_constant": 60,
    "ranked_results": [
      { "chunk_id": "chunk_4521", "dense_rank": 1, "sparse_rank": 3, "fused_score": 0.0421 },
      { "chunk_id": "chunk_990", "dense_rank": 5, "sparse_rank": 1, "fused_score": 0.0398 }
    ]
  }
}
```

- **Why RRF over naive score averaging:** Dense (cosine similarity) and sparse (BM25) scores live on different, incomparable scales. RRF only needs each result's *rank* within its own list, not the raw score — sidestepping the scale-mismatch problem entirely. This is worth explaining unprompted; it's a strong signal of understanding the underlying math, not just calling a library function.

### 4.3 Re-Ranking Contract

```json
{
  "rerank_request": {
    "query_id": "q_4521",
    "candidates": ["chunk_4521", "chunk_990", "chunk_2103"],
    "reranker_model": "cross-encoder-v2",
    "top_n_output": 5
  }
}
```

- **Principal-level note:** Re-ranking with a cross-encoder is more accurate than the initial retrieval's similarity score because it jointly encodes the query and document, but it's too slow to run over the full corpus — the two-stage pattern (cheap broad retrieval → expensive precise re-ranking on a small candidate set) is the standard production answer, and naming it as a deliberate two-stage funnel (not "I added re-ranking") is the stronger framing.

---

## 5. Production Failure Modes & Mitigations

### 5.1 Stale Document Retrieval

**Symptom:** The system confidently cites information from a superseded document version alongside the current one.

```json
{
  "freshness_check": {
    "chunk_id": "chunk_4521",
    "document_version": "2",
    "current_document_version": "3",
    "is_stale": true,
    "action": "exclude_from_retrieval"
  }
}
```

- **Fix:** Filter by document version/freshness at retrieval time, and separately, run a deprecation sweep that removes superseded chunks from the index rather than relying on filtering alone to catch every case.

### 5.2 Retrieval-Generation Mismatch (Hallucination Despite Correct Retrieval)

**Symptom:** The correct document was retrieved, but the model still produces an answer not actually grounded in it.

```json
{
  "groundedness_check": {
    "response_id": "resp_881",
    "claims_extracted": 3,
    "claims_supported_by_context": 2,
    "claims_unsupported": 1,
    "groundedness_score": 0.67,
    "action": "flag_for_review"
  }
}
```

- **Fix:** This is a prompt/generation problem, not a retrieval problem — diagnose by checking groundedness explicitly rather than assuming better retrieval will fix it. Often the fix is instructing the model more strictly to answer only from provided context and say so explicitly when context is insufficient.

### 5.3 Context Window Dilution

**Symptom:** Retrieval returns many marginally relevant chunks, diluting the genuinely relevant one's influence in a crowded context window.

- **Fix:** Token-aware context packing prioritized by relevance score (not just inclusion by rank cutoff), and consider whether fewer, more targeted chunks outperform more, broader ones — this is an empirical question per use case, not a universal default.

### 5.4 Cross-Tenant Data Leakage in Shared Indexes

**Symptom:** A multi-tenant RAG system occasionally surfaces another tenant's content.

```json
{
  "tenant_isolation_audit": {
    "query_tenant": "tenant_acme",
    "retrieved_chunk_tenant": "tenant_globex",
    "isolation_violation": true,
    "enforcement_layer": "application_filter_only"
  }
}
```

- **Fix:** Enforce tenant isolation at the index/query level (a hard filter in the vector search call itself), not only in application-layer post-filtering — application-layer-only filtering is one bug away from exactly this leak, and `enforcement_layer` in the audit record above is the field that reveals the architectural gap.

### 5.5 Embedding Model Drift After Silent Upgrade

**Symptom:** Retrieval quality degrades after a provider silently updates an embedding model behind the same API endpoint.

- **Fix:** Pin embedding model versions explicitly where the provider allows it; if not, monitor retrieval quality metrics continuously so a silent upgrade is caught by a metric drop, not by waiting for user complaints.

---

## 6. Complexity Reduction for RAG Specifically

Applying the orchestration document's general complexity-reduction discipline to RAG specifically:

| Degree of Freedom | Reduction Strategy |
|---|---|
| Number of retrieval sources | Default to one primary index; add fallback sources only with a defined trigger condition (Section 2.2) |
| Chunking strategies | Pick one default strategy per content type (prose vs. table vs. code); don't let it vary per-document without a documented reason |
| Embedding models in flight | One embedding model per index; re-embed the full index on model upgrade rather than mixing versions |
| Retrieval hop count | Hard cap (Section 2.4) |
| Context packing logic | One deterministic packing algorithm, not query-dependent ad hoc logic |

---

## 7. Decision Framework

Before adding RAG sophistication, ask:

1. Is the current failure actually a retrieval problem, or a generation/groundedness problem? (Section 5.2 — diagnose before reaching for more retrieval machinery.)
2. Does this query type genuinely need multi-hop, or would better single-pass retrieval solve it more cheaply?
3. Is the corpus actually large/messy enough to need hybrid search, or would tuned dense retrieval alone suffice?
4. Have you measured retrieval precision/recall independently of end-to-end answer quality? (You can't fix what you haven't isolated.)
5. Is staleness or access control a real risk for this corpus — and if so, is it enforced at the index level or only hoped for at the application level?

**The governing test, same as orchestration:** every retrieval should be explainable (what was retrieved and why), reproducible (same query, same index version, same result), and auditable (full provenance trail). If you can't answer "what did the model actually see" for a given bad answer, you don't have production RAG — you have a demo with a vector database attached.

---

## Companion Documents

Part of the Principal AI Engineer / FDE architecture series:

- `Agent_Orchestration_Architecture.md` — patterns and schemas this file builds on (multi-hop retrieval as planner-executor, complexity reduction discipline)
- `Model_Serving_Architecture_Deep_Dive.md` — serving the embedding and generation models this pipeline depends on
- `Fine_Tuning_Workflow_Architecture.md` — when to fine-tune vs. rely on RAG alone
- `IAM_ZeroTrust_Agent_Architecture.md` — the access-control enforcement referenced in Section 5.4
- `AI_Governance_Compliance_Schemas.md` — provenance requirements for regulated RAG deployments
- `Observability_Evaluation_Architecture.md` — the groundedness and retrieval-quality evaluation infrastructure referenced throughout
