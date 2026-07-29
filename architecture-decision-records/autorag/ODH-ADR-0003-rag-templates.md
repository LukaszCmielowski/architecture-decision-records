# Open Data Hub - AutoRAG RAG Templates

|                |            |
| -------------- | ---------- |
| Date           | 2026-07-29 |
| Scope          | AutoRAG Component |
| Status         | Proposed |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | TBD |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) |

## What

This ADR defines AutoRAG / ai4rag RAG templates: the reusable retrieve-and-generate blueprints parameterized during optimization. It covers the shipping simple RAG template, the proposed Relationship-Enriched path, planned Graph RAG sub-templates, and pattern deployment implications.

## Why

Template choice determines composition (retrieve/generate), infrastructure (vector store vs graph DB), optimization knobs, and deployment contracts. An ADR records which templates ship, which are proposed, and how engines will be compared before a Graph RAG product default is chosen.

## Goals

* Document the current simple retrieve-then-generate template and its optimization / inference contract
* Define Relationship-Enriched RAG as a Milvus-only path that validates relational extraction before graph infrastructure
* Document peer Graph RAG options (LightRAG core and Neo4j GraphRAG), comparison criteria, and open engine decision
* Describe pattern-to-deployment mapping (starter-kit today; OpenShift-native later)

## Non-Goals

* Selecting the final Graph RAG GA engine in this ADR (decision remains open pending leaderboard comparison)
* Detailed pipeline parameter tables for simple-RAG presets (see ODH-ADR-0002)
* Full pattern.json field reference (see ODH-ADR-0004)

## How

A **RAG template** is the reusable workflow blueprint AutoRAG / ai4rag parameterizes during optimization. GAM explores values inside a template (chunking, retrieval, generation, prompts); the template itself defines **how** retrieve and generate are composed.

## Table of contents

- [Overview](#overview)
- [Current RAG template](#current-rag-template)
- [Relationship-Enriched RAG template](#relationship-enriched-rag-template)
  - [How it works](#how-it-works)
  - [Enrichment strategies](#enrichment-strategies)
  - [Extraction output format](#extraction-output-format)
  - [Search-space integration](#search-space-integration)
  - [Extraction model requirements](#extraction-model-requirements)
- [Graph RAG template](#graph-rag-template)
  - [Shared prerequisites](#shared-prerequisites)
  - [Sub-template: LightRAG core](#sub-template-lightrag-core)
  - [Sub-template: Neo4j GraphRAG](#sub-template-neo4j-graphrag)
  - [LangChain / LangGraph (optional orchestration)](#langchain--langgraph-optional-orchestration)
  - [Pipeline integration](#pipeline-integration)
  - [Pattern artifacts](#pattern-artifacts)
  - [Comparison](#comparison)
    - [Where LightRAG sits](#where-lightrag-sits)
    - [Value of LightRAG vs Relationship-Enriched RAG](#value-of-lightrag-vs-relationship-enriched-rag)
    - [Value of LightRAG vs Neo4j GraphRAG](#value-of-lightrag-vs-neo4j-graphrag)
    - [Decision sketch](#decision-sketch)
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
| **Current (simple) RAG** | Shipping | Single retrieve → generate hop (`SimpleRAG`) | Milvus |
| **Relationship-Enriched RAG** | Proposed | Simple RAG + relation-enriched chunks via LightRAG extraction | Milvus + extraction LLM (no graph DB) |
| **Graph RAG** | Planned / future | Knowledge-graph RAG with **two peer sub-templates** (decision open): LightRAG core, or Neo4j GraphRAG (+ optional LangChain/LangGraph orchestration) | Neo4j (required); LightRAG also uses PostgreSQL for KV/vector/doc-status |

Optimized instances of a template are emitted as **RAG patterns** (`pattern.json`). See [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md).

Pipeline stage: **`rag_templates_optimization`** in the documents RAG optimization managed pipeline. Template selection is expected via `rag_template_type` (`simple` | `lightrag` | `neo4j_graphrag` | `both` for comparative runs). Which Graph RAG engine becomes the product default is **not decided yet** — both paths remain in scope for evaluation on the same pipeline `test_data` / leaderboard.

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
| **Generation** | One LLM call with system / user / context templates over retrieved chunks |
| **Inference export** | Frozen **`inference.responses_template`** for OGX `POST /v1/responses` (`file_search` + generation) |
| **Optimization** | GAM over chunking × embedding × retrieval × generation (and optional [prompt tuning](./ODH-ADR-0006-prompt-tuning.md) candidates) |

**Naming note:** ai4rag `hybrid` means dense + ranker fusion over chunks. That is **not** the same as LightRAG’s `hybrid` query mode (local + global graph — see [LightRAG core](#sub-template-lightrag-core)).

What this template does **not** include today: multi-hop retrieval, tool-calling loops, entity/knowledge-graph traversal, or agent planners. Evaluation uses the metrics in [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md).

---

## Relationship-Enriched RAG template

A pragmatic middle ground between simple vector RAG and full Graph RAG. Instead of requiring a graph database, this template uses LightRAG's entity/relation extraction to **enrich existing chunks with relational metadata** before indexing — similar in spirit to how Docling `contextualize()` adds structural metadata, but for semantic/relational context.

This approach validates whether relationship extraction improves retrieval quality on the existing leaderboard **before** committing to graph-database infrastructure.

### How it works

```text
documents → text_extraction (Docling)
               │
               ├── chunking (existing) ────────────────────────┐
               │                                               │
               └── LightRAG extract_entities() ──┐             │
                    (entity/relation extraction)  │             │
                                                  ▼             ▼
                                        Relationship enrichment
                                        (attach to chunks as metadata)
                                                  │
                                                  ▼
                                   ┌──────────────────────────────┐
                                   │  Enriched chunk:             │
                                   │    text: "original chunk"    │
                                   │    metadata:                 │
                                   │      entities: [...]         │
                                   │      relationships: [...]    │
                                   │      relation_keywords:      │
                                   │        ["collaboration", …]  │
                                   │    embed_text (optional):    │
                                   │      "original chunk         │
                                   │       [Entities: A, B]       │
                                   │       [Relations: A→B …]"    │
                                   └──────────────────────────────┘
                                                  │
                                                  ▼
                                   Existing vector store (Milvus)
                                   Existing retriever (vector/hybrid)
                                   Existing generator
```

The extraction LLM runs **once during indexing** (same as Docling parsing), not per query. Enriched chunks flow through the existing `file_search` + generation inference path — no new `inference.responses_template` is needed.


TODO: which LLM if more than 1 ---> simple metric to select one 
TODO: param to include or NOT (to be used as extra) --> chunker: 

```
chunk_enrichment: {
   "relationships": {
      "enable": "true",
      "include": true,
      "model_id": "llama_3_1_70B"
   },
   "structure": {
      "enable": "true",
      "include": "true"
   },
   "contextual": {
      "enable": "true",
      "include": true,
      "model_id": "llama_3_1_70B"
   }
}
```
### Enrichment strategies

Two strategies can be explored independently or combined:

| Strategy | Mechanism | Retrieval impact |
|----------|-----------|------------------|
| **A — Metadata keys** (filter/boost) | Store extracted entity names, types, and relationship keywords as filterable metadata fields in the vector store | At query time, extract entities from the question, then filter or boost chunks sharing entities or relationship types. Works with any vector store supporting metadata filtering (Milvus does). |
| **B — Contextual embedding enrichment** | Prepend or append entity/relationship summaries to the chunk text *before* embedding (analogous to Docling `contextualize()`) | The embedding model captures relational context, improving semantic similarity for relational queries. Controlled by a single `include_relations` boolean, same pattern as `include_metadata`. |

Strategy B parallels the existing `include_metadata` flow — call it `enrich_with_relations()` or `relationalize()`. Strategy A adds entity-based signals alongside the existing hybrid (vector + BM25) retrieval. Both compose: `include_metadata: true` + `include_relations: true` = structural + relational context.

### Extraction output format

LightRAG's `extract_entities()` produces structured JSON for each chunk:

```json
{
  "entities": [
    {
      "entity_name": "AutoRAG",
      "entity_type": "SYSTEM",
      "entity_description": "Automated RAG optimization system within RHOAI"
    }
  ],
  "relationships": [
    {
      "src_id": "AutoRAG",
      "tgt_id": "Kubeflow Pipelines",
      "relationship_description": "AutoRAG uses Kubeflow Pipelines for workflow orchestration",
      "relationship_strength": 9,
      "relationship_keywords": ["orchestration", "dependency"]
    }
  ]
}
```

Key fields: `relationship_strength` (numeric importance score) and `relationship_keywords` (semantic tags like "collaboration", "dependency") enable filtering and ranking at retrieval time without graph traversal.

### Search-space integration

Enrichment adds new dimensions to the existing GAM search space without changing the template class:

| Parameter | Type | Role |
|-----------|------|------|
| `include_relations` | `bool` | Enable relationship enrichment (Strategy B) — analogous to `include_metadata` |
| `relation_filter_mode` | `none` / `boost` / `filter` | Strategy A — how entity metadata affects retrieval scoring |
| `extraction_model_id` | `str` | OGX model used for entity/relation extraction |

These sit alongside existing dimensions (`chunking.method`, `include_metadata`, `search_mode`, `number_of_chunks`) and are evaluated on the same leaderboard metrics.

| Concern | Detail |
|---------|--------|
| **ai4rag type** | `SimpleRAG` (same template class, extended search space) |
| **pattern_type** | `simple` (enrichment is a pre-indexing concern, not a new pattern type) |
| **Infrastructure** | No graph DB — only the existing Milvus vector store + an extraction LLM call during indexing |
| **Cost** | One extraction LLM pass per document at index time; no per-query overhead |

### Extraction model requirements

LightRAG's entity/relation extraction is LLM-driven and requires a model with strong structured-output and reasoning capabilities. Community guidance suggests ≥ 32B parameters for reliable extraction. Use `extraction_model_id` to select a stronger OGX-served model when available — this can differ from the generation model.

**What Enriched RAG does not cover:** multi-hop graph traversal, global/thematic queries (LightRAG `global` mode), or deep graph reasoning. Those use cases still require a graph-native template (see [Graph RAG template](#graph-rag-template)).

---

## Graph RAG template

**Graph RAG** is a planned template family for multi-hop and relational questions where one-shot vector retrieval underperforms. It sits beside simple RAG inside the same Kubeflow DAG (Path A: ai4rag templates on `BaseRAGTemplate`), not as a chunk-only `BaseVectorStore` path.

Two **peer sub-templates** are documented. Engine choice for GA is **open**; both can be implemented and compared on the shared leaderboard (`faithfulness`, `answer_correctness`, cost/latency).

```text
question
   │
   ▼
┌────────────────────────────────────────────────────────────┐
│  Graph RAG (sub-template selected at run time)             │
│                                                            │
│   LightRAG core              │   Neo4j GraphRAG            │
│   (Postgres + Neo4j)          │   (Neo4j-only hybrid)       │
│            └─────────────────┴──────────┘                  │
│                          ▼                                 │
│              Knowledge graph (+ vectors per path)          │
│                         [Neo4j]                            │
└────────────────────────────────────────────────────────────┘
   │
   ▼
 answer (+ graph / hybrid context)
```

| Concern | Direction |
|---------|-----------|
| **Store** | Neo4j required for the graph; LightRAG also uses Postgres for vector/KV/doc-status |
| **Indexing** | `extracted_text` → entity/relation extraction → graph (+ vector) write; indexes are **not** shared with simple-RAG Milvus chunk collections |
| **Retrieval** | Dual-level graph modes (LightRAG) or Neo4j-native vector/hybrid/Cypher retrievers — not a single `file_search` hop |
| **Generation** | LLM grounded in graph/hybrid context; OGX-backed embed/LLM adapters |
| **Optimization** | Initially fixed configs; later GAMOpt over mode / retriever type / `top_k` / chunk tokens / schema constraints |
| **Export** | Durable `pattern.json` with `pattern_type`, storage profile, workspace — reconnect after pod exit; richer than today’s `responses_template` alone |

Graph RAG remains **out of the current Tech Preview template set** until productized in ai4rag and pipelines-components.

### Shared prerequisites

| Item | Notes |
|------|--------|
| **Graph store** | Neo4j (`neo4j_secret_name`) — required for both Graph RAG sub-templates |
| **PostgreSQL** | Required for the LightRAG path (`postgres_secret_name`) for KV, vector, and doc-status storage (alongside Neo4j for the graph) |
| **OGX** | Embedding, generation, and extraction model ids via OGX secrets — no hardcoded vendor SDKs |
| **Corpus** | Same Docling → `extracted_text` and pipeline `test_data` as simple RAG |
| **vs simple RAG** | Graph RAG adds template types; it does not replace the current vector/`file_search` path |

Without a reachable Neo4j, Graph RAG templates cannot run.

**Naming note:** LightRAG `hybrid` query mode (local + global graph) is **not** the same as ai4rag simple-RAG `hybrid` (dense + BM25 + RRF). See [LightRAG core](#sub-template-lightrag-core) for mode definitions.

### Sub-template: LightRAG core

[LightRAG](https://github.com/HKUDS/LightRAG) ([Programming with Core](https://github.com/HKUDS/LightRAG/blob/main/docs/ProgramingWithCore.md)) — dual-level graph + vector indexing with built-in insert and query modes.

| Layer | Role |
|-------|------|
| **LightRAG core** | Entity/relation extraction, graph+vector indexing, dual-level retrieval, incremental updates |
| **Neo4j** (`Neo4JStorage`) | Knowledge graph |
| **PostgreSQL** (+ `vector`) | `PGVectorStorage`, `PGKVStorage`, `PGDocStatusStorage` |
| **Orchestration** | LightRAG’s own retrieve/generate loop (not LangChain-centric) |
| **OGX** | `EmbeddingFunc` / LLM callbacks; prefer a stronger model for extraction when available |

**Storage profile:** `ogx_pgvector_neo4j` (default) — four logical roles on **two** physical DBs (KV cannot be skipped; it colocates on Postgres). PVC is scratch only; patterns reference Postgres workspace + Neo4j labels.

```python
# Profile: ogx_pgvector_neo4j (Neo4j + Postgres)
LightRAG(
    working_dir=WORKING_DIR,  # scratch only
    vector_storage="PGVectorStorage",
    kv_storage="PGKVStorage",
    doc_status_storage="PGDocStatusStorage",
    graph_storage="Neo4JStorage",
)
```

```text
docs (extracted_text) ──▶ LightRAG insert ──▶ Postgres + Neo4j
question ──▶ mode (naive|local|global|hybrid|mix) ──▶ context ──▶ generate ──▶ answer
```

| Concern | Detail |
|---------|--------|
| **Query modes** | `naive`, `local` (entity/relation), `global` (thematic), `hybrid` (local+global graph), **`mix`** (KG + chunk vectors) |
| **Naming** | LightRAG `hybrid` ≠ ai4rag simple-RAG `hybrid` |
| **Initial config** | Fixed `mode=mix` is a practical starting point; later GAMOpt: `mode`, `top_k`, `chunk_top_k`, chunk tokens (~1200), `enable_rerank`, model ids |
| **Tuning defaults (when optimizing)** | `mix` + rerank if available + `top_k≈20` + larger chunks + low temperature (~0.2) + `enable_llm_cache` |
| **ai4rag type** | `LightRAGTemplate` |
| **pattern_type** | `lightrag` |

Later storage profiles (same template, Neo4j graph retained): `ogx_milvus_neo4j` (scale), `dev_local` (CI).

| Storage profile | Graph store | Physical DBs | LightRAG | Neo4j GraphRAG |
|-----------------|-------------|--------------|----------|----------------|
| `ogx_pgvector_neo4j` | Neo4j | 2 (Postgres + Neo4j) | Yes | No (different sub-template) |
| `neo4j_hybrid_only` | Neo4j | 1 (Neo4j) | No | Yes |

### Sub-template: Neo4j GraphRAG

Official [`neo4j-graphrag`](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html) for KG construction and Neo4j-native retrieval, then `GraphRAG(retriever=..., llm=...).search`. This path can stand alone for indexing + retrieval; LangChain/LangGraph is **optional** on top (see below).

| Layer | Role |
|-------|------|
| **KG build** | `SimpleKGPipeline` / `GraphSchema` from `extracted_text` (+ vector and full-text indexes in Neo4j) |
| **Retrievers** | See table below |
| **Generation** | `neo4j_graphrag.generation.GraphRAG` |
| **Neo4j** | Single DB for graph + vector + full-text — profile `neo4j_hybrid_only` |
| **OGX** | Adapters implementing Neo4j `LLM` / `Embedder` interfaces |

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
| **Storage** | Neo4j-only; do **not** unify storage layouts with LightRAG until an engine is chosen for GA |
| **Initial config** | Fixed retriever + `top_k` / retrieval Cypher; later GAMOpt: retriever type, Cypher depth/LIMIT, schema constraints |
| **ai4rag type** | `Neo4jGraphRAGTemplate` |
| **pattern_type** | `neo4j_graphrag` |
| **pattern fields** | `storage_profile: neo4j_hybrid_only`, `retriever_type` (`hybrid_cypher` \| `vector_cypher` \| `text2cypher`), Neo4j index names; embedding model locked at index time |
| **Deps** | `neo4j-graphrag`, `neo4j`; optional `langgraph` for agentic orchestration |

Prefer native Neo4j retrievers over reimplementing the same patterns with LangChain `LLMGraphTransformer` + `Neo4jVector` + custom structured retrievers.

### LangChain / LangGraph (optional orchestration)

Applies primarily to the **Neo4j GraphRAG** path. LangChain chat models are an optional LLM adapter into Neo4j `GraphRAG` — retrieval itself does not require LangChain.

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
documents_discovery → text_extraction → search_space_preparation
                                   ↓
             rag_templates_optimization
             (SimpleRAG | LightRAGTemplate | Neo4jGraphRAGTemplate)
                                   ↓
                        leaderboard_evaluation
                   (pattern_type: simple | lightrag | neo4j_graphrag)
```

| Layer | Change |
|-------|--------|
| **ai4rag** | Templates on `BaseRAGTemplate`; `AI4RAGExperiment.rag_template_type`; OGX bridges; optional extras (`lightrag-hku`, `neo4j-graphrag`) |
| **KFP** | `rag_template_type`, storage secrets, graph search-space dims, leaderboard `pattern_type` columns |
| **Path A** | Preferred — templates inside the existing documents RAG optimization DAG |
| **Path B** | Standalone KFP (`lightrag_indexing` → `lightrag_evaluation`) only if Path A merge is blocked |
| **Template contract** | `build_index` / `generate` / `generate_stream` — not chunk-only vector store |
| **Key params** | `rag_template_type`, `storage_profile`, `ogx_secret_name`, `embedding_model_id`, `generation_model_id`, `extraction_model_id`, `postgres_secret_name`, `neo4j_secret_name`, `query_mode` / `retriever_type`, `index_workspace` |
| **Acceptance** | Reconnect via `pattern.json` after pod exit; scores on pipeline `test_data` |
| **Out of scope (v1)** | Sharing simple-RAG chunk collections as LightRAG indexes; LightRAG Server/WebUI; multimodal RAG-Anything; LangChain-first Neo4j retrieval; MiniRAG |

### Pattern artifacts

LightRAG pattern sketch:

```json
{
  "pattern_type": "lightrag",
  "workspace": "<stable-index-id>",
  "storage": {
    "storage_profile": "ogx_pgvector_neo4j",
    "kv_storage": "PGKVStorage",
    "vector_storage": "PGVectorStorage",
    "doc_status_storage": "PGDocStatusStorage",
    "graph_storage": "Neo4JStorage"
  },
  "query_defaults": { "mode": "mix", "top_k": 40 },
  "indexing": { "embedding_model_id": "<ogx>", "chunk_token_size": 1200 },
  "generation": { "model_id": "<ogx>" },
  "metrics": { "faithfulness": 0.0, "answer_correctness": 0.0 }
}
```

Neo4j GraphRAG patterns use `pattern_type: neo4j_graphrag`, `storage_profile: neo4j_hybrid_only`, `retriever_type`, and Neo4j index names, with the same metrics contract for leaderboard comparison.

### Comparison

Three relationship-aware options sit on a spectrum from **reuse existing Milvus** → **full dual-level Graph RAG** → **Neo4j-native Graph RAG**. LangChain/LangGraph is **not** a peer engine — optional orchestration on the Neo4j path only.

#### Where LightRAG sits

| | **Relationship-Enriched RAG** | **LightRAG core** | **Neo4j GraphRAG** |
|--|------------------------------|-------------------|--------------------|
| **Intent** | Prove extraction helps retrieval **without** a graph DB | Own dual-level KG+vector RAG (local / global / mix) | Own Neo4j-native hybrid + Cypher RAG |
| **ai4rag type** | `SimpleRAG` (extended search space) | `LightRAGTemplate` | `Neo4jGraphRAGTemplate` |
| **pattern_type** | `simple` | `lightrag` | `neo4j_graphrag` |
| **Physical DBs (typical)** | 1 — Milvus (existing) | **2** — Postgres + Neo4j (`ogx_pgvector_neo4j`) | **1** — Neo4j only |
| **Logical stores** | Chunk vectors (+ metadata) | **4 roles:** KV + vector + doc-status + graph (KV cannot be skipped) | Graph + vector + full-text **inside Neo4j** |
| **Python deps** | LightRAG extract (or equivalent) only at index | `lightrag-hku` + Postgres drivers + Neo4j | `neo4j-graphrag` + `neo4j` (+ optional `langgraph`) |
| **Ops surface** | Same as simple RAG + extraction LLM | Heaviest default: two DB secrets, workspace, four storage backends | One graph DB; fewer moving parts than default LightRAG |
| **Multi-hop / thematic** | No (metadata filter/boost only) | Yes (`local` / `global` / `hybrid` / `mix`) | Yes (Cypher neighborhood, Text2Cypher, ToolsRetriever) |
| **Reuse simple-RAG index** | Yes | No — separate LightRAG index | No — separate Neo4j indexes |
| **Integration effort** | Low | Medium (storage profiles + reconnect) | Low–medium |

#### Value of LightRAG vs Relationship-Enriched RAG

Enriched RAG **borrows** LightRAG’s `extract_entities()` but never persists a traversable graph.

| | Pros | Cons |
|--|------|------|
| **Enriched** | No new DB; stays on Milvus + existing `file_search` export; cheap way to A/B extraction on today’s leaderboard | No graph traversal, no `global`/`mix` modes; relational signal capped at metadata / embedding text |
| **LightRAG** | True multi-hop and thematic retrieval; incremental graph+vector index; query modes tuned for entity vs corpus-level questions | Extraction cost **plus** graph/vector infra; indexes not shared with simple RAG; heavier secrets and pattern reconnect |

**Takeaway:** Enriched RAG answers “does extraction help?” LightRAG answers “do we need graph-native retrieval?” Prefer Enriched first if Milvus-only ops is a hard constraint.

#### Value of LightRAG vs Neo4j GraphRAG

Both are peer Graph RAG engines; GA default is **open**. Difference is **stack shape**, not “graph vs no graph.”

| Dimension | **LightRAG** | **Neo4j GraphRAG** |
|-----------|--------------|--------------------|
| **Strength** | Dual-level modes (`naive`→`mix`) and entity/relation vector indexes out of the box; thin insert/query API | Single DB (`neo4j_hybrid_only`); schema-guided KG; first-class Hybrid/Cypher/Text2Cypher retrievers; vendor-supported `neo4j-graphrag` |
| **Dependency weight** | **Heavier:** Postgres (KV+vector+doc-status) **and** Neo4j; four logical backends; `lightrag-hku` surface area | **Lighter ops:** one Neo4j; fewer storage classes; deps centered on `neo4j` / `neo4j-graphrag` |
| **Storage variety** | Profiles: `ogx_pgvector_neo4j`, later `ogx_milvus_neo4j` / `dev_local` (Neo4j graph retained) | Low — Neo4j only (`neo4j_hybrid_only`) |
| **Flexibility** | Optional future Milvus vector role alongside Neo4j | Best when Neo4j is already (or will be) platform standard |
| **Risk** | Community stack; more failure modes (two DBs, KV required) | Couples product to Neo4j; less “mode” vocabulary than LightRAG; Text2Cypher quality depends on schema + LLM |

**Takeaway:** Choose **LightRAG** when dual-level query modes matter more than minimizing DB count. Choose **Neo4j GraphRAG** when one graph database, Cypher-native retrievers, and a smaller dependency footprint matter more than LightRAG’s mode set.

#### Decision sketch

```text
Need relation signal but Milvus-only?
  └─ Yes → Relationship-Enriched RAG

Need multi-hop / thematic graph RAG? (Neo4j required)
  ├─ Prefer one DB + Neo4j Cypher retrievers → Neo4j GraphRAG
  ├─ Prefer dual-level modes (Postgres + Neo4j) → LightRAG
  └─ Unsure → run both (`rag_template_type=both`) on the same test_data / leaderboard
```

**Eval:** Enriched, LightRAG, and Neo4j GraphRAG compete on the same metrics (`faithfulness`, `answer_correctness`, cost/latency). Product default for Graph RAG remains open until comparative runs close the gap. Frontier ideas (path-ranked context, cost models such as LazyGraphRAG) can apply to either graph path without forcing an early engine pick.

### Optimization parameters by template

AutoRAG / GAM should explore knobs that change **retrieval or generation quality** without exploding index-rebuild cost. Prefer a **phased** search space: fix heavy index settings first, then optimize query-time dims; widen later once baselines exist. Full simple-RAG dims: [experiment settings](./ODH-ADR-0002-experiment-settings.md).

| Template | Optimize (high value) | Optimize later / conditional | Usually **fix** (not GAM dims) |
|----------|----------------------|------------------------------|----------------------------------|
| **Current (simple) RAG** | `chunking.method` / `chunk_size` / `chunk_overlap`; `include_metadata` (hybrid); `embedding_model_id`; `search_mode`; `number_of_chunks`; hybrid `ranker_*`; `generation_model_id` + gen params (temp, max tokens); optional [prompt tuning](./ODH-ADR-0006-prompt-tuning.md) candidates | Ranker strategy variants; embedding allow-list size; prompt candidate count | `vector_io_provider_id`, Docling preset (`speed`/`balanced`), corpus / `test_data`, OGX endpoint |
| **Relationship-Enriched RAG** | Everything in simple RAG **plus** `include_relations`, `relation_filter_mode` (`none`/`boost`/`filter`), `extraction_model_id` | Combining A+B strategies; relation keyword boost weight | Same infra as simple RAG; do not treat extraction as a per-query dim |
| **LightRAG core** | **Query-time:** `mode` (`naive`/`local`/`global`/`hybrid`/`mix`), `top_k`, `chunk_top_k`, `enable_rerank`; **gen:** `generation_model_id`, temperature (~0.1–0.3) | **Index-time (costly):** `chunk_token_size` (~800–1500), `extraction_model_id`, embedding model (locks vectors) | `storage_profile` (`ogx_pgvector_neo4j`, …), DB secrets, workspace id, `enable_llm_cache` (ops, not quality objective) |
| **Neo4j GraphRAG** | **Query-time:** `retriever_type` (`hybrid_cypher` / `vector_cypher` / `text2cypher` / …), `top_k`, Cypher hop depth / `LIMIT`; **gen:** model + temperature | **Index-time:** `GraphSchema` strictness / entity types; embedding model; full-text vs vector index params | `storage_profile: neo4j_hybrid_only`, Neo4j secret, LangGraph on/off (orchestration, not core RAG score) |

**Shared rules**

| Rule | Why |
|------|-----|
| Index-time dims (chunk size, embed model, extraction model, KG schema) **rebuild** the store — sample sparsely or as an outer loop | Dominates wall-clock and $ |
| Query-time dims (`mode` / `retriever_type`, `top_k`, rerank, gen temp) are cheap to sweep on a frozen index | Best GAM ROI |
| Embedding / extraction model ids are categorical and expensive — small allow-lists | Avoid combinatorial blow-up |
| Do not GAM-optimize storage backend choice in the same trial as quality knobs | Confounds quality with infra; compare profiles in separate runs |

**Suggested v1 Graph RAG search spaces (frozen index, then query+gen):**

```text
LightRAG:     mode ∈ {mix, hybrid, local, global}  ×  top_k ∈ {10,20,40}  ×  enable_rerank ∈ {false,true}  ×  gen temp
Neo4j GraphRAG: retriever_type ∈ {hybrid_cypher, vector_cypher}  ×  top_k  ×  cypher_depth  ×  gen temp
(+ optional outer: extraction_model_id or chunk_token_size / schema — few values only)
```

Start Graph RAG with **fixed** `mode=mix` or `retriever_type=hybrid_cypher` for smoke tests; turn on GAM once reconnect + leaderboard work.

---

## RAG application deployment

Once a RAG pattern is optimized and the index is built, the pattern must be served behind a **production inference endpoint**. Today this uses the **agentic-starter-kit** deployment model; future work will extend this to OpenShift-native deployment.

### Current: starter-kit deployment

The [`agentic-starter-kits`](https://github.com/red-hat-data-services/agentic-starter-kits) repository provides a LangGraph-based [Agentic RAG template](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) that wraps retrieval and generation into a deployable agent application.

```text
Optimized pattern (pattern.json)
   │
   ├── settings (chunking, embedding, retrieval, generation)
   │
   ├── inference.responses_template ──────────► OGX POST /v1/responses
   │     (simple RAG: file_search + generation)     (platform API — existing)
   │
   └── Agentic RAG starter-kit ──────────────► Deployed agent application
         (LangGraph orchestration)                  (container on OpenShift)
```

The starter-kit agent is a **LangGraph** application that:

| Concern | Detail |
|---------|--------|
| **Orchestration** | LangGraph graph: route query → retrieve → generate → respond |
| **Retrieval** | Vector store via OGX (`file_search` / `vector_io`) — Milvus (local) or pgvector (OpenShift) |
| **Generation** | LLM via OGX (`BASE_URL`, `MODEL_ID`) — same model from the pattern's `generation.model_id` |
| **API** | `POST /chat/completions` (streaming / non-streaming), `GET /health` |
| **Configuration** | Environment variables sourced from `pattern.json` settings: `MODEL_ID`, `EMBEDDING_MODEL`, `EMBEDDING_DIMENSION`, `VECTOR_STORE_ID`, `VECTOR_STORE_PROVIDER` |
| **Deployment** | Helm chart → OpenShift (`make deploy`); local dev via `make run-app` |
| **Observability** | Optional MLflow tracing (`MLFLOW_TRACKING_URI`) aligned with AutoML/AutoRAG tracking model |

**Pattern → starter-kit wiring:** the pattern's `settings` and `inference` blocks provide the configuration values the starter-kit consumes via environment variables. Today this wiring is manual (copy values to `.env` or `values.yaml`); the goal is to automate it so that selecting a pattern in the Dashboard pre-fills the deployment form.

| Pattern field | Starter-kit env var |
|---------------|---------------------|
| `settings.generation.model_id` | `MODEL_ID` |
| `settings.embedding.model_id` | `EMBEDDING_MODEL` |
| `settings.embedding.embedding_params.embedding_dimension` | `EMBEDDING_DIMENSION` |
| `settings.vector_store_binding.vector_store_id` | `VECTOR_STORE_ID` |
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
- **Graph RAG agent templates** — LangGraph agents that use LightRAG query modes or Neo4j Cypher retrievers instead of simple `file_search`
- **Scaling and lifecycle** — horizontal pod autoscaling, health probes, rolling updates tied to re-indexed patterns

### Template-to-deployment mapping

Each RAG template produces a pattern with a different inference contract. The deployment mechanism must match:

| Template | Inference contract | Starter-kit path | Notes |
|----------|-------------------|-------------------|-------|
| **Simple RAG** | `inference.responses_template` → OGX `POST /v1/responses` with `file_search` | [`agentic_rag`](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) template | Shipping today |
| **Enriched RAG** | Same as simple RAG (enrichment is in the indexed chunks, transparent at query time) | Same `agentic_rag` template | No agent-side changes needed |
| **LightRAG core** | LightRAG query API (`mode`, `top_k`) → graph + vector context → generate | New starter-kit template or `agentic_rag` extended with LightRAG retriever | Requires LightRAG client in agent, Neo4j + Postgres connectivity |
| **Neo4j GraphRAG** | Neo4j retriever (`HybridCypherRetriever`, etc.) → `GraphRAG.search` → answer | New starter-kit template with `neo4j-graphrag` retrievers + LangGraph orchestration | Requires Neo4j connectivity; LangGraph orchestration natural fit |

**Simple and Enriched RAG** use the existing `agentic_rag` starter-kit unchanged — the `inference.responses_template` and OGX `file_search` contract are the same. **Graph RAG templates** require new agent templates that replace `file_search` with graph-native retrieval (LightRAG query modes or Neo4j Cypher retrievers).

---

## Related

- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md) — search-space dimensions for the current template
- [Prompt tuning](./ODH-ADR-0006-prompt-tuning.md) — optional prompt candidates injected into the current template
- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md) — pattern artifacts and Responses export
- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md) — benchmark metrics (`faithfulness`, `answer_correctness`, …)
- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
- [LightRAG](https://github.com/HKUDS/LightRAG) / [Programming with Core](https://github.com/HKUDS/LightRAG/blob/main/docs/ProgramingWithCore.md)
- [neo4j-graphrag — User Guide: RAG](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html)
- [neo4j-graphrag — KG Builder](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_kg_builder.html)
- Optional orchestration: [LangGraph + Neo4j](https://neo4j.com/blog/developer/neo4j-graphrag-workflow-langchain-langgraph/)
- [Agentic starter-kits](https://github.com/red-hat-data-services/agentic-starter-kits) — deployment templates for RAG agents
- [Agentic RAG template](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) — LangGraph-based RAG agent (current deployment path)
