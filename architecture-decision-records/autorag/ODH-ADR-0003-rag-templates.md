# Open Data Hub - AutoRAG RAG Templates

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-26 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-2357](https://redhat.atlassian.net/browse/RHAISTRAT-2357) · [RHAIRFE-2709](https://redhat.atlassian.net/browse/RHAIRFE-2709) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) · [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0007-maas-support](./ODH-ADR-0007-maas-support.md) |

## What

This ADR defines AutoRAG / ai4rag RAG templates: the reusable retrieve-and-generate blueprints parameterized during optimization. It covers the shipping simple RAG template, planned Neo4j Graph RAG, and pattern deployment implications.

## Why

Template choice determines composition (retrieve/generate), infrastructure (vector store vs graph DB), optimization knobs, and deployment contracts. An ADR records which templates ship, which are proposed, and that Graph RAG is **Neo4j-native** (`neo4j-graphrag`).

## Goals

* Document the current simple retrieve-then-generate template and its optimization / inference contract
* Document Neo4j Graph RAG as the planned graph template (single engine)
* Describe pattern-to-deployment mapping (starter-kit today; OpenShift-native later)

## Non-Goals

* Relationship-Enriched RAG (vector-store-only entity/relation chunk enrichment) — removed from scope
* Alternate Graph RAG engines or dual-engine bake-offs
* Detailed pipeline parameter tables for simple-RAG presets (see ODH-ADR-0002)
* Full pattern.json field reference (see ODH-ADR-0004)

## How

A **RAG template** is the reusable workflow blueprint AutoRAG / ai4rag parameterizes during optimization. GAM explores values inside a template (chunking, retrieval, generation, prompts); the template itself defines **how** retrieve and generate are composed.

## Table of contents

- [Overview](#overview)
- [Current RAG template](#current-rag-template)
- [Graph RAG template](#graph-rag-template)
  - [Prerequisites](#prerequisites)
  - [Neo4j GraphRAG](#neo4j-graphrag)
  - [LangChain / LangGraph (optional orchestration)](#langchain--langgraph-optional-orchestration)
  - [Pipeline integration](#pipeline-integration)
  - [Pattern artifacts](#pattern-artifacts)
  - [Optimization parameters by template](#optimization-parameters-by-template)
- [RAG application deployment](#rag-application-deployment)
  - [Current: starter-kit deployment](#current-starter-kit-deployment)
  - [Future: OpenShift-native deployment](#future-openshift-native-deployment)
  - [Template-to-deployment mapping](#template-to-deployment-mapping)
- [Related](#related)

---

## Overview

| Template | Status | Composition | Infrastructure |
|----------|--------|-------------|----------------|
| **Current (simple) RAG** | Shipping | Single retrieve → generate hop (`SimpleRAG`) | Milvus or PGVector |
| **Graph RAG** | Planned / future | Neo4j-native KG + hybrid/Cypher retrieval (`neo4j-graphrag`; optional LangGraph orchestration) | Neo4j (graph + vector + full-text) |

Optimized instances of a template are emitted as **RAG patterns** (`pattern.json`). See [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md).

Pipeline stage: **`rag_templates_optimization`** in the documents RAG optimization managed pipeline. Template selection is expected via `rag_template_type` (`simple` | `neo4j_graphrag`).

---

## Current RAG template

The shipping AutoRAG / ai4rag template is a **single-turn retrieve-then-generate** pipeline (often called simple / standard RAG):

```text
question
   │
   ▼
┌──────────────┐     top-k chunks      ┌──────────────┐
│  Retriever   │ ───────────────────▶  │  Generator   │ ──▶ answer
│  (vector /   │                       │  (LLM +      │
│   hybrid)    │                       │   prompts)   │
└──────────────┘                       └──────────────┘
```

| Concern | Behavior |
|---------|----------|
| **Indexing** | Chunk → embed → write vector store ([experiment settings](./ODH-ADR-0002-experiment-settings.md)) |
| **Retrieval** | One query; `number_of_chunks`, `search_mode` (`vector` / `keyword` / `hybrid`), optional ranker |
| **Generation** | One LLM call with `system_message_text` / `user_message_text` / `context_template_text` over retrieved chunks |
| **Inference export** | `pattern.json` `settings` — LangChain retrieval plus MaaS `/v1/chat/completions` assembled from `settings.generation` |
| **Optimization** | GAM over chunking × embedding × retrieval × generation (and optional [prompt tuning](./ODH-ADR-0006-prompt-tuning.md) candidates) |

**Naming note:** ai4rag `hybrid` means dense + BM25 fused with RRF (or weighted fusion) over chunks. That is **not** Neo4j `HybridRetriever` / `HybridCypherRetriever` (vector + full-text, optionally plus Cypher).

What this template does **not** include today: multi-hop retrieval, tool-calling loops, entity/knowledge-graph traversal, or agent planners. Evaluation uses the metrics in [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md).

---

## Graph RAG template

**Graph RAG** is a planned template for multi-hop and relational questions where one-shot vector retrieval underperforms. It sits beside simple RAG inside the same Kubeflow DAG (Path A: ai4rag templates on `BaseRAGTemplate`), not as a chunk-only `BaseVectorStore` path.

The engine is **Neo4j GraphRAG** (`neo4j-graphrag`): schema-guided KG construction, Neo4j-native vector/hybrid/Cypher retrievers, and `GraphRAG.search`. LangChain/LangGraph is **optional** orchestration on top, not a peer engine.

```text
question
   │
   ▼
┌────────────────────────────────────────────────────────────┐
│  Graph RAG (Neo4j GraphRAG)                                │
│                                                            │
│   SimpleKGPipeline / GraphSchema                           │
│              ▼                                             │
│   Neo4j (graph + vector + full-text)                       │
│              ▼                                             │
│   HybridCypherRetriever (or VectorCypher / Text2Cypher)    │
│              ▼                                             │
│   GraphRAG.search                                          │
└────────────────────────────────────────────────────────────┘
   │
   ▼
 answer (+ graph / hybrid context)
```

| Concern | Direction |
|---------|-----------|
| **Store** | Neo4j only (graph + vector + full-text); profile `neo4j_hybrid_only` |
| **Indexing** | `extracted_text` → entity/relation extraction → Neo4j write; indexes are **not** shared with simple-RAG chunk collections |
| **Retrieval** | Neo4j-native vector / hybrid / Cypher retrievers — not a single vector hop |
| **Generation** | LLM grounded in graph/hybrid context; MaaS-backed embed/LLM adapters |
| **Optimization** | Initially fixed configs; later GAMOpt over retriever type / `top_k` / Cypher depth / schema constraints |
| **Export** | Durable `pattern.json` in the [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) envelope; Neo4j knobs live under `settings.retrieval` / `settings.vector_store_binding` — reconnect after pod exit |

Graph RAG remains **out of the current Tech Preview template set** until productized in ai4rag and pipelines-components.

### Prerequisites

| Item | Notes |
|------|--------|
| **Graph store** | Neo4j (`graph_db_secret_name`) |
| **MaaS** | Embedding, generation, and extraction model ids via `maas_secret_name` — no hardcoded vendor SDKs |
| **Corpus** | Same Docling → `extracted_text` and pipeline `test_data` as simple RAG |
| **vs simple RAG** | Graph RAG adds a template type; it does not replace the current LangChain vector retrieve + MaaS generate path |

Without a reachable Neo4j Connection (`graph_db_secret_name`), Graph RAG cannot run. Secret keys: [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#connections).

### Neo4j GraphRAG

Official [`neo4j-graphrag`](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html) for KG construction and Neo4j-native retrieval, then `GraphRAG(retriever=..., llm=...).search`. This path can stand alone for indexing + retrieval; LangChain/LangGraph is **optional** on top (see below).

| Layer | Role |
|-------|------|
| **KG build** | `SimpleKGPipeline` / `GraphSchema` from `extracted_text` (+ vector and full-text indexes in Neo4j) |
| **Retrievers** | See table below |
| **Generation** | `neo4j_graphrag.generation.GraphRAG` |
| **Neo4j** | Single DB for graph + vector + full-text — profile `neo4j_hybrid_only` |
| **MaaS** | Adapters implementing Neo4j `LLM` / `Embedder` interfaces |

| Retriever | Role |
|-----------|------|
| `VectorRetriever` | Vector ANN |
| `VectorCypherRetriever` | Vector hit → Cypher neighborhood |
| `HybridRetriever` | Vector + full-text |
| `HybridCypherRetriever` | Hybrid + Cypher traversal (strong default candidate) |
| `Text2CypherRetriever` | NL → Cypher |
| `ToolsRetriever` | Route across retrievers |

```text
docs ──▶ SimpleKGPipeline / GraphSchema ──▶ Neo4j (graph + vector + full-text)
question ──▶ HybridCypherRetriever (or VectorCypher / Text2Cypher / …)
                └── GraphRAG.search ──▶ answer
```

| Concern | Detail |
|---------|--------|
| **Storage** | Neo4j-only (`neo4j_hybrid_only`) |
| **Initial config** | Fixed retriever + `top_k` / retrieval Cypher; later GAMOpt: retriever type, Cypher depth/LIMIT, schema constraints |
| **ai4rag type** | `Neo4jGraphRAGTemplate` |
| **pattern.json** | Same envelope as [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md); `settings.retrieval.method` is a Neo4j retriever (`hybrid_cypher` \| `vector_cypher` \| `text2cypher`); `settings.vector_store_binding.provider_type` is `neo4j`; embedding model locked at index time |
| **Deps** | `neo4j-graphrag`, `neo4j`; optional `langgraph` for agentic orchestration |

Prefer native Neo4j retrievers over reimplementing the same patterns with LangChain `LLMGraphTransformer` + `Neo4jVector` + custom structured retrievers.

### LangChain / LangGraph (optional orchestration)

LangChain chat models are an optional LLM adapter into Neo4j `GraphRAG` — retrieval itself does not require LangChain.

| Need | Use |
|------|-----|
| Neo4j indexing + retrieval | **`neo4j-graphrag` only** |
| Multi-step agents, HITL, tools outside Neo4j | **LangGraph** calling Neo4j retrievers |
| Org-wide LangChain / LangSmith standard | Thin LangChain LLM wrapper into Neo4j `GraphRAG` |

```text
Track: Neo4jGraphRAGTemplate + neo4j-graphrag retrievers
         │
         └── optional LangGraph router / multi-tool agents
```

### Pipeline integration

```text
documents_discovery → search_space_preparation → text_extraction
                                   ↓
             models_pre_selector
                                   ↓
             rag_templates_optimization
             (SimpleRAG | Neo4jGraphRAGTemplate)
                                   ↓
                        leaderboard (built in rag_templates_optimization)
                   (pattern_type: simple | neo4j_graphrag)
```

| Layer | Change |
|-------|--------|
| **ai4rag** | Templates on `BaseRAGTemplate`; `AI4RAGExperiment.rag_template_type`; MaaS bridges; extra `neo4j-graphrag` |
| **KFP** | `rag_template_type`, `graph_db_secret_name`, graph search-space dims, leaderboard `pattern_type` columns |
| **Path A** | Preferred — template inside the existing documents RAG optimization DAG |
| **Template contract** | `build_index` / `generate` / `generate_stream` — not chunk-only vector store |
| **Key params** | `rag_template_type`, `storage_profile`, `maas_secret_name`, `embedding_model_id`, `generation_model_id`, `extraction_model_id`, `graph_db_secret_name`, `retriever_type` |
| **Acceptance** | Reconnect via `pattern.json` after pod exit; scores on pipeline `test_data` |
| **Out of scope (v1)** | Sharing simple-RAG chunk collections as Neo4j indexes; LangChain-first Neo4j retrieval (use `neo4j-graphrag` retrievers) |

### Pattern artifacts

Neo4j GraphRAG pattern sketch (same top-level envelope as simple RAG; extra knobs under `settings`):

```json
{
  "name": "PatternGraph1",
  "settings": {
    "vector_store_binding": {
      "provider_type": "neo4j",
      "collection_name": "<neo4j-index>"
    },
    "retrieval": {
      "method": "hybrid_cypher",
      "number_of_chunks": 10
    },
    "generation": {
      "model_id": "<maas>",
      "temperature": 0.2,
      "max_completion_tokens": 2048,
      "system_message_text": "<system>",
      "user_message_text": "<user>",
      "context_template_text": "<context>"
    }
  },
  "evaluation": { "metrics": [] },
  "indexing": { "pipeline_spec": {} }
}
```

Graph RAG patterns use the same `evaluation` / `indexing` contract as simple RAG for leaderboard comparison. Query-time retriever choice is `settings.retrieval.method` (for example `hybrid_cypher`).

### Optimization parameters by template

AutoRAG / GAM should explore knobs that change **retrieval or generation quality** without exploding index-rebuild cost. Prefer a **phased** search space: fix heavy index settings first, then optimize query-time dims; widen later once baselines exist. Full simple-RAG dims: [experiment settings](./ODH-ADR-0002-experiment-settings.md).

| Template | Optimize (high value) | Optimize later / conditional | Usually **fix** (not GAM dims) |
|----------|----------------------|------------------------------|----------------------------------|
| **Current (simple) RAG** | `chunking.method` / `chunk_size` / `chunk_overlap`; `include_metadata` (hybrid); `embedding_model_id`; `search_mode`; `number_of_chunks`; hybrid `ranker_*`; `generation_model_id` + gen params (temp, max tokens); optional [prompt tuning](./ODH-ADR-0006-prompt-tuning.md) candidates | Ranker strategy variants; embedding allow-list size; prompt candidate count | `vector_db_secret_name`, Docling preset (`speed`/`balanced`), corpus / `test_data`, MaaS Connection |
| **Neo4j Graph RAG** | **Query-time:** `retriever_type` (`hybrid_cypher` / `vector_cypher` / `text2cypher` / …), `top_k`, Cypher hop depth / `LIMIT`; **gen:** model + temperature | **Index-time:** `GraphSchema` strictness / entity types; embedding model; full-text vs vector index params | `storage_profile: neo4j_hybrid_only`, `graph_db_secret_name`, LangGraph on/off (orchestration, not core RAG score) |

**Shared rules**

| Rule | Why |
|------|-----|
| Index-time dims (chunk size, embed model, extraction model, KG schema) **rebuild** the store — sample sparsely or as an outer loop | Dominates wall-clock and $ |
| Query-time dims (`retriever_type`, `top_k`, rerank, gen temp) are cheap to sweep on a frozen index | Best GAM ROI |
| Embedding / extraction model ids are categorical and expensive — small allow-lists | Avoid combinatorial blow-up |
| Do not GAM-optimize storage backend choice in the same trial as quality knobs | Confounds quality with infra |

**Suggested v1 Graph RAG search space (frozen index, then query+gen):**

```text
Neo4j GraphRAG: retriever_type ∈ {hybrid_cypher, vector_cypher}  ×  top_k  ×  cypher_depth  ×  gen temp
(+ optional outer: extraction_model_id or schema — few values only)
```

Start Graph RAG with **fixed** `retriever_type=hybrid_cypher` for smoke tests; turn on GAM once reconnect + leaderboard work.

---

## RAG application deployment

Once a RAG pattern is optimized and the index is built, the pattern must be served behind a **production inference endpoint**. Today this uses the **agentic-starter-kit** deployment model; future work will extend this to OpenShift-native deployment.

### Current: starter-kit deployment

The [`agentic-starter-kits`](https://github.com/red-hat-data-services/agentic-starter-kits) repository provides a LangGraph-based [Agentic RAG template](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) that wraps retrieval and generation into a deployable agent application.

```text
Optimized pattern (pattern.json + agentic_template.zip)
   │
   ├── settings.retrieval + vector_store_binding ──► LangChain retrieve
   │
   ├── settings.generation ────────────────────────► MaaS /v1/chat/completions
   │     (system / user / context templates)
   │
   └── agentic_template.zip ───────────────────────► Deployed agent application
         (parameterized agentic RAG starter-kit)         (container on OpenShift)
```

The starter-kit agent is a **LangGraph** application that:

| Concern | Detail |
|---------|--------|
| **Orchestration** | LangGraph graph: route query → retrieve → generate → respond |
| **Retrieval** | LangChain vector store (`vector_db_secret_name`) — Milvus or PGVector |
| **Generation** | LLM via MaaS (`MAAS_BASE_URL`, `MAAS_API_KEY`) — same model from the pattern's `generation.model_id` |
| **API** | `POST /chat/completions` (streaming / non-streaming), `GET /health` |
| **Configuration** | Environment variables sourced from `pattern.json` `settings`: `MODEL_ID`, `EMBEDDING_MODEL`, `EMBEDDING_DIMENSION`, collection / provider |
| **Deployment** | Helm chart → OpenShift (`make deploy`); local dev via `make run-app` |
| **Observability** | Optional MLflow tracing (`MLFLOW_TRACKING_URI`) aligned with AutoML/AutoRAG tracking model |

**Pattern → starter-kit wiring:** `rag_templates_optimization` emits **`agentic_template.zip`** per pattern — the [agentic RAG starter-kit](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) parameterized from `pattern.json` `settings`. Dashboard / Studio can unpack the zip; a manual `.env` / `values.yaml` copy remains a fallback.

| Pattern field | Starter-kit env var |
|---------------|---------------------|
| `settings.generation.model_id` | `MODEL_ID` |
| `settings.generation.system_message_text` | (agent system prompt) |
| `settings.generation.user_message_text` | (agent user prompt template) |
| `settings.generation.context_template_text` | (chunk formatting) |
| `settings.embedding.model_id` | `EMBEDDING_MODEL` |
| `settings.embedding.embedding_params.embedding_dimension` | `EMBEDDING_DIMENSION` |
| `settings.vector_store_binding.collection_name` | collection / `VECTOR_STORE_ID` |
| `settings.vector_store_binding.provider_type` | `VECTOR_STORE_PROVIDER` |
| `settings.retrieval.number_of_chunks` | (agent config / retriever `top_k`) |

### Future: OpenShift-native deployment

The starter-kit model serves as the initial deployment path. Future work will extend this to **OpenShift-native deployment** where optimized RAG patterns are deployed as first-class platform workloads:

| Phase | Deployment model | Status |
|-------|------------------|--------|
| **Current** | Starter-kit: manual pattern → `.env` → `make deploy` / Helm chart | Available |
| **Next** | Dashboard-driven: select pattern → auto-populate deployment form → deploy agent | Planned |
| **Future** | OpenShift-native: pattern-driven operator or pipeline that provisions the agent, vector store binding, and route as managed resources | Future |

Key capabilities to add:
- **Automated pattern-to-deployment wiring** — Dashboard reads `pattern.json`, pre-fills deployment config
- **Graph RAG agent templates** — LangGraph agents that use Neo4j Cypher retrievers instead of simple vector retrieval
- **Scaling and lifecycle** — horizontal pod autoscaling, health probes, rolling updates tied to re-indexed patterns

### Template-to-deployment mapping

Each RAG template produces a pattern with a different inference contract. The deployment mechanism must match:

| Template | Inference contract | Starter-kit path | Notes |
|----------|-------------------|-------------------|-------|
| **Simple RAG** | `settings.generation` → MaaS `/v1/chat/completions` + LangChain retrieval | [`agentic_rag`](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) via **`agentic_template.zip`** | Shipping today |
| **Neo4j Graph RAG** | Neo4j retriever (`HybridCypherRetriever`, etc.) → `GraphRAG.search` → answer | New starter-kit template with `neo4j-graphrag` retrievers + LangGraph orchestration | Requires Neo4j connectivity; LangGraph orchestration natural fit |

**Simple RAG** uses the existing `agentic_rag` starter-kit — retrieve from `settings.vector_store_binding` and generate from `settings.generation`. **Graph RAG** requires a new agent template that replaces vector-only retrieval with Neo4j Cypher retrievers.

---

## Related

- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md) — search-space dimensions for the current template
- [Prompt tuning](./ODH-ADR-0006-prompt-tuning.md) — optional prompt candidates injected into the current template
- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md) — `pattern.json` schema and retrieve / generate
- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md) — benchmark metrics (`faithfulness`, `answer_correctness`, …)
- [MaaS support](./ODH-ADR-0007-maas-support.md) — MaaS and vector-DB Connections
- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
- [neo4j-graphrag — User Guide: RAG](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html)
- [neo4j-graphrag — KG Builder](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_kg_builder.html)
- Optional orchestration: [LangGraph + Neo4j](https://neo4j.com/blog/developer/neo4j-graphrag-workflow-langchain-langgraph/)
- [Agentic starter-kits](https://github.com/red-hat-data-services/agentic-starter-kits) — deployment templates for RAG agents
- [Agentic RAG template](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) — LangGraph-based RAG agent (current deployment path)
