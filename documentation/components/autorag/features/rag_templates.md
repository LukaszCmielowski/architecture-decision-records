# RAG templates

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
  - [Neo4j-free deployment (PostgreSQL AGE)](#neo4j-free-deployment-postgresql-age)
  - [LangChain / LangGraph (optional orchestration)](#langchain--langgraph-optional-orchestration)
  - [Pipeline integration](#pipeline-integration)
  - [Pattern artifacts](#pattern-artifacts)
  - [Comparison](#comparison)
    - [Where LightRAG sits](#where-lightrag-sits)
    - [Value of LightRAG vs Relationship-Enriched RAG](#value-of-lightrag-vs-relationship-enriched-rag)
    - [Value of LightRAG vs Neo4j GraphRAG](#value-of-lightrag-vs-neo4j-graphrag)
    - [Decision sketch](#decision-sketch)
- [Related](#related)

---

## Overview

| Template | Status | Composition | Infrastructure |
|----------|--------|-------------|----------------|
| **Current (simple) RAG** | Shipping | Single retrieve → generate hop (`SimpleRAG`) | Milvus |
| **Relationship-Enriched RAG** | Proposed | Simple RAG + relation-enriched chunks via LightRAG extraction | Milvus + extraction LLM (no graph DB) |
| **Graph RAG** | Planned / future | Knowledge-graph RAG with **two peer sub-templates** (decision open): LightRAG core, or Neo4j GraphRAG (+ optional LangChain/LangGraph orchestration) | Neo4j and/or PostgreSQL AGE |

Optimized instances of a template are emitted as **RAG patterns** (`pattern.json`). See [RAG pattern inference](./rag_pattern_inference.md).

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
| **Indexing** | Chunk → embed → write vector store ([experiment settings](./experiment_settings.md)) |
| **Retrieval** | One query; `number_of_chunks`, `search_mode` (`vector` / `keyword` / `hybrid`), optional ranker |
| **Generation** | One LLM call with system / user / context templates over retrieved chunks |
| **Inference export** | Frozen **`inference.responses_template`** for OGX `POST /v1/responses` (`file_search` + generation) |
| **Optimization** | GAM over chunking × embedding × retrieval × generation (and optional [prompt tuning](./prompt_tuning.md) candidates) |

**Naming note:** ai4rag `hybrid` means dense + ranker fusion over chunks. That is **not** the same as LightRAG’s `hybrid` query mode (local + global graph — see [LightRAG core](#sub-template-lightrag-core)).

What this template does **not** include today: multi-hop retrieval, tool-calling loops, entity/knowledge-graph traversal, or agent planners. Evaluation uses the metrics in [RAG pattern evaluation](./rag_pattern_evaluation.md).

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
│   (Postgres + Neo4j/AGE)     │   (Neo4j-only hybrid)       │
│            └─────────────────┴──────────┘                  │
│                          ▼                                 │
│              Knowledge graph (+ vectors per path)          │
│              [Neo4j  or  PostgreSQL AGE]                   │
└────────────────────────────────────────────────────────────┘
   │
   ▼
 answer (+ graph / hybrid context)
```

| Concern | Direction |
|---------|-----------|
| **Store** | Graph store required (Neo4j or PostgreSQL AGE); LightRAG also uses Postgres for vector/KV/doc-status |
| **Indexing** | `extracted_text` → entity/relation extraction → graph (+ vector) write; indexes are **not** shared with simple-RAG Milvus chunk collections |
| **Retrieval** | Dual-level graph modes (LightRAG) or Neo4j-native vector/hybrid/Cypher retrievers — not a single `file_search` hop |
| **Generation** | LLM grounded in graph/hybrid context; OGX-backed embed/LLM adapters |
| **Optimization** | Initially fixed configs; later GAMOpt over mode / retriever type / `top_k` / chunk tokens / schema constraints |
| **Export** | Durable `pattern.json` with `pattern_type`, storage profile, workspace — reconnect after pod exit; richer than today’s `responses_template` alone |

Graph RAG remains **out of the current Tech Preview template set** until productized in ai4rag and pipelines-components.

### Shared prerequisites

| Item | Notes |
|------|--------|
| **Graph store** | Neo4j (`neo4j_secret_name`) **or** PostgreSQL with the [AGE extension](https://age.apache.org/) — see [Neo4j-free deployment](#neo4j-free-deployment-postgresql-age). Neo4j GraphRAG sub-template requires Neo4j; LightRAG supports either. |
| **PostgreSQL** | Required for the LightRAG path (`postgres_secret_name`) for KV, vector, and doc-status storage. When using AGE, the same Postgres instance also serves as the graph store. |
| **OGX** | Embedding, generation, and extraction model ids via OGX secrets — no hardcoded vendor SDKs |
| **Corpus** | Same Docling → `extracted_text` and pipeline `test_data` as simple RAG |
| **vs simple RAG** | Graph RAG adds template types; it does not replace the current vector/`file_search` path |

Without a reachable graph store (Neo4j or PostgreSQL AGE), Graph RAG templates cannot run.

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

**Storage profile:** `ogx_pgvector_neo4j` (default) — four logical roles on **two** physical DBs (KV cannot be skipped; it colocates on Postgres). PVC is scratch only; patterns reference Postgres workspace + Neo4j labels. For Neo4j-free deployment see [`postgres_all_in_one_age`](#neo4j-free-deployment-postgresql-age).

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

Later storage profiles (same template, different backends): `ogx_milvus_neo4j` (scale), `dev_local` (CI). For `postgres_all_in_one_age` see below.

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

### Neo4j-free deployment (PostgreSQL AGE)

The LightRAG sub-template can run **without Neo4j** by replacing `Neo4JStorage` with PostgreSQL and the [AGE extension](https://age.apache.org/) (Apache AGE — openCypher on relational tables). All four LightRAG storage roles collapse onto a **single Postgres instance**.

```python
# Profile: postgres_all_in_one_age (Postgres-only, no Neo4j)
LightRAG(
    working_dir=WORKING_DIR,
    vector_storage="PGVectorStorage",
    kv_storage="PGKVStorage",
    doc_status_storage="PGDocStatusStorage",
    graph_storage="AGEStorage",
)
```

| Concern | Detail |
|---------|--------|
| **Storage profile** | `postgres_all_in_one_age` |
| **Physical DBs** | 1 (PostgreSQL with `vector` + `age` extensions) |
| **Graph queries** | openCypher via AGE — compatible with LightRAG's graph traversal modes |
| **When to use** | Neo4j unavailable or undesired; simpler ops (one DB to manage); environments where graph query depth ≤ 2–3 hops suffices |
| **Trade-offs** | Deep graph traversals may be slower than Neo4j-native; AGE extension must be installed on the Postgres instance; Neo4j GraphRAG sub-template (`neo4j-graphrag` retrievers) is **not** compatible with this profile |
| **ai4rag type** | `LightRAGTemplate` (same template, different `storage_profile`) |
| **pattern_type** | `lightrag` |

This profile also enables a **relational alternative**: extract entities and relationships via LightRAG's `extract_entities()`, store them in standard relational tables (entities, relationships, chunks with embeddings), and build a custom SQL-based retriever that uses JOINs instead of Cypher traversals. This approach suits teams with strong Postgres expertise and avoids both Neo4j and the AGE extension dependency.

| Storage profile | Graph store | Physical DBs | LightRAG compatible | Neo4j GraphRAG compatible |
|-----------------|-------------|-------------|---------------------|--------------------------|
| `ogx_pgvector_neo4j` | Neo4j | 2 (Postgres + Neo4j) | Yes | No (different sub-template) |
| `postgres_all_in_one_age` | PostgreSQL AGE | 1 (Postgres) | Yes | No |
| `neo4j_hybrid_only` | Neo4j | 1 (Neo4j) | No | Yes |

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
| **Physical DBs (typical)** | 1 — Milvus (existing) | **2** — Postgres + Neo4j (`ogx_pgvector_neo4j`), or **1** — Postgres+AGE | **1** — Neo4j only |
| **Logical stores** | Chunk vectors (+ metadata) | **4 roles:** KV + vector + doc-status + graph (KV cannot be skipped) | Graph + vector + full-text **inside Neo4j** |
| **Python deps** | LightRAG extract (or equivalent) only at index | `lightrag-hku` + Postgres drivers + Neo4j **or** AGE | `neo4j-graphrag` + `neo4j` (+ optional `langgraph`) |
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
| **Strength** | Dual-level modes (`naive`→`mix`) and entity/relation vector indexes out of the box; thin insert/query API; can drop Neo4j via AGE | Single DB (`neo4j_hybrid_only`); schema-guided KG; first-class Hybrid/Cypher/Text2Cypher retrievers; vendor-supported `neo4j-graphrag` |
| **Dependency weight** | **Heavier by default:** Postgres (KV+vector+doc-status) **and** Neo4j, or Postgres+AGE+`vector`; four logical backends; `lightrag-hku` surface area | **Lighter ops:** one Neo4j; fewer storage classes; deps centered on `neo4j` / `neo4j-graphrag` |
| **Storage variety** | High — profiles span `ogx_pgvector_neo4j`, `postgres_all_in_one_age`, later `ogx_milvus_neo4j` / `dev_local` | Low — Neo4j only; incompatible with AGE / LightRAG layouts |
| **Flexibility** | Swap graph backend (Neo4j ↔ AGE); optional future Milvus vector role | Best when Neo4j is already (or will be) platform standard; no first-class Neo4j-free path |
| **Risk** | Community stack; more failure modes (two DBs, KV required); AGE deep hops may lag Neo4j | Couples product to Neo4j; less “mode” vocabulary than LightRAG; Text2Cypher quality depends on schema + LLM |

**Takeaway:** Choose **LightRAG** when dual-level query modes and/or Neo4j-optional (AGE) deployment matter more than minimizing DB count. Choose **Neo4j GraphRAG** when one graph database, Cypher-native retrievers, and a smaller dependency footprint matter more than LightRAG’s mode set.

#### Decision sketch

```text
Need relation signal but Milvus-only?
  └─ Yes → Relationship-Enriched RAG

Need multi-hop / thematic graph RAG?
  ├─ Prefer one DB + Neo4j Cypher retrievers → Neo4j GraphRAG
  ├─ Prefer dual-level modes / AGE (no Neo4j) → LightRAG
  └─ Unsure → run both (`rag_template_type=both`) on the same test_data / leaderboard
```

**Eval:** Enriched, LightRAG, and Neo4j GraphRAG compete on the same metrics (`faithfulness`, `answer_correctness`, cost/latency). Product default for Graph RAG remains open until comparative runs close the gap. Frontier ideas (path-ranked context, cost models such as LazyGraphRAG) can apply to either graph path without forcing an early engine pick.


---

## Related

- [AutoRAG optimization settings](./experiment_settings.md) — search-space dimensions for the current template
- [Prompt tuning](./prompt_tuning.md) — optional prompt candidates injected into the current template
- [RAG pattern inference](./rag_pattern_inference.md) — pattern artifacts and Responses export
- [RAG pattern evaluation](./rag_pattern_evaluation.md) — benchmark metrics (`faithfulness`, `answer_correctness`, …)
- [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- [LightRAG](https://github.com/HKUDS/LightRAG) / [Programming with Core](https://github.com/HKUDS/LightRAG/blob/main/docs/ProgramingWithCore.md)
- [neo4j-graphrag — User Guide: RAG](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html)
- [neo4j-graphrag — KG Builder](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_kg_builder.html)
- Optional orchestration: [LangGraph + Neo4j](https://neo4j.com/blog/developer/neo4j-graphrag-workflow-langchain-langgraph/)
- [Apache AGE](https://age.apache.org/) — openCypher on PostgreSQL (Neo4j-free graph storage for LightRAG)
