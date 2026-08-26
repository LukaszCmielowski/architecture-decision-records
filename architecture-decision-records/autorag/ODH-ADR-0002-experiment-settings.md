# Open Data Hub - AutoRAG Optimization Settings

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-26 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1673](https://redhat.atlassian.net/browse/RHAISTRAT-1673) · [RHAISTRAT-2040](https://redhat.atlassian.net/browse/RHAISTRAT-2040) · [RHAISTRAT-2440](https://redhat.atlassian.net/browse/RHAISTRAT-2440) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) · [ODH-ADR-0007-maas-support](./ODH-ADR-0007-maas-support.md) |

## What

This ADR documents the AutoRAG documents RAG optimization pipeline public parameters, quality presets, and the chunking / retrieval search-space dimensions explored by ai4rag during GAM optimization.

## Why

Pipeline operators and Dashboard integrations need a stable contract for how optimization runs are configured (data sources, MaaS and vector-DB Connections, required model lists, presets) and which chunking and retrieval knobs form the searchable configuration space. Capturing this as an ADR keeps the contract versioned alongside the AutoRAG architecture.

## Goals

* Define the public parameter surface of `documents_rag_optimization_pipeline`
* Document speed and balanced presets (Docling behavior, chunking envelope, inference concurrency)
* Specify chunking methods (recursive, hybrid) and retrieval modes (vector, keyword, hybrid) used in the search space

## Non-Goals

* RAG template composition beyond the current simple-RAG search space (see ODH-ADR-0003)
* pattern.json inference / indexing export schema (see ODH-ADR-0004)
* Evaluation metric backends and judge selection (see ODH-ADR-0005)
* MaaS integration rationale (see ODH-ADR-0007)

## How

Pipeline parameters, presets, and the **chunking / retrieval** search space for **`documents_rag_optimization_pipeline`** ([opendatahub-io/pipelines-components](https://github.com/opendatahub-io/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline)). ADR context: [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md). Inference and vector-store wiring: [ODH-ADR-0007](./ODH-ADR-0007-maas-support.md).

Search-space preparation runs **before** text extraction so unresponsive or misconfigured MaaS models fail the experiment before Docling work.

## Table of contents

- [Pipeline parameters](#pipeline-parameters)
- [Connections](#connections)
- [Presets](#presets)
- [Chunking methods](#chunking-methods)
- [Retrieval methods](#retrieval-methods)
- [Related](#related)

---

## Pipeline parameters

Public surface of [`pipeline.py`](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py) (verify on your **pipelines-components** tag):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `test_data_secret_name` | `str` | (required) | Secret for S3-compatible test-data access (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`; `AWS_DEFAULT_REGION` optional) |
| `test_data_bucket_name` | `str` | (required) | Bucket containing the evaluation benchmark JSON |
| `test_data_key` | `str` | (required) | Object key of the test data file |
| `input_data_secret_name` | `str` | (required) | Secret for document corpus access (same key convention) |
| `input_data_bucket_name` | `str` | (required) | Bucket containing source documents |
| `input_data_key` | `str` | `""` | Object key or prefix for input documents |
| `maas_secret_name` | `str` | (required) | MaaS Connection (`MAAS_BASE_URL`, `MAAS_API_KEY`). Used for model validation, embeddings, generation, and judge calls. |
| `vector_db_secret_name` | `str` | (required) | Vector DB Connection. Key prefix selects the backend: `MILVUS_*` (at least `MILVUS_URI`) or `PGVECTOR_*`. |
| `embedding_models` | `list[str]` | (required) | Non-empty list of embedding model ids for the search space. MaaS does not expose type metadata, so models cannot be inferred. |
| `generation_models` | `list[str]` | (required) | Non-empty list of generation / foundation model ids for the search space. Same reason as `embedding_models`. |
| `optimization_metric` | `str` | `overall_score` | GAM objective: `faithfulness`, `answer_correctness`, `context_correctness`, `answer_relevance`, `overall_score`. The first three are deterministic Unitxt metrics. `answer_relevance` (LLM judge) is always computed; it drives ranking only when selected, or as part of `overall_score`. |
| `optimization_max_rag_patterns` | `int` | `8` | Max patterns to evaluate and retain (`max_number_of_rag_patterns`) |
| `preset` | `str` | `speed` | Quality tier — maps to Docling extraction, chunking search space, contextual enrichment, and inference concurrency ([Presets](#presets)) |

---

## Connections

Secrets are Kubernetes Secret **names** (RHOAI Connections). Values are injected as environment variables; they are not pipeline-parameter payloads.

**MaaS** (`maas_secret_name`)

| Key | Role |
|-----|------|
| `MAAS_BASE_URL` | OpenAI-compatible MaaS gateway base URL |
| `MAAS_API_KEY` | API key for discovery, embeddings, chat, and judge |

Mounted on `search_space_preparation`, `models_pre_selector`, and `rag_templates_optimization`.

**Vector database** (`vector_db_secret_name`)

Backend is selected from keys present on the secret (mounted on `rag_templates_optimization`):

| Backend | Keys | Selection |
|---------|------|-----------|
| Milvus | `MILVUS_URI` (required), `MILVUS_TOKEN`, `MILVUS_SERVER_CERT` | Any `MILVUS*` env var |
| PGVector | `PGVECTOR_HOST`, `PGVECTOR_PORT`, `PGVECTOR_DB`, `PGVECTOR_USER`, `PGVECTOR_PASSWORD` | Else any `PGVECTOR*` env var |

Missing both prefixes fails the optimization step.

**Object storage** (`test_data_secret_name`, `input_data_secret_name`)

`documents_discovery` maps input vs test credentials to `INPUT_DATA_AWS_*` and `TEST_DATA_AWS_*`. `text_extraction` uses unprefixed `AWS_*` from the input-data secret.

---

## Presets

Each preset fixes **ingestion, chunking search-space envelope, contextual enrichment, and inference concurrency**. ai4rag still optimizes embedding, retrieval, and generation (from the required model lists) within that envelope. Both presets use the **same KFP resource tier**.

| Preset | Chunking methods | Chunk sizes | Chunk overlaps | Docling PDF tables | Contextual enrichment | `inference_max_threads` |
|--------|------------------|-------------|----------------|--------------------|-----------------------|-------------------------|
| `speed` (default) | `recursive` only | `128`, `256`, `512` | `32`, `64` | `do_table_structure: false` | none | 10 |
| `balanced` | `recursive` + `hybrid` | `512`, `1024`, `2048` | `0`, `128`, `256` | `do_table_structure: true` ([TableFormer](https://docling-project.github.io/docling/guides/pdf-processing/)) | LLM contextual enrichment on hybrid trials | 4 |

`inference_max_threads` is the benchmark query concurrency in `rag_templates_optimization` (`balanced` is lower because per-request context is larger).

**Shared resource envelope** (not preset-specific):

| Task | CPU / memory request | CPU / memory limit |
|------|----------------------|--------------------|
| `documents_discovery`, `search_space_preparation`, `models_pre_selector` | 2 / 8Gi | 32 / 64Gi |
| `text_extraction`, `rag_templates_optimization` | 4 / 16Gi | 32 / 64Gi |
| `publish_component_stage_map` | 0.5 / 512Mi | 1 / 1Gi |

Docling extraction (`text_extraction`) is fixed per run by the preset, not repeated in each `pattern.json`. Use `speed` for plain text or quick runs; `balanced` for structured PDFs/DOCX with tables, headings, and LLM contextual enrichment.

---

## Chunking methods

Chunking splits documents into embeddable segments. **`chunking`** fields control boundaries and Docling serialization (including optional structural metadata via `include_metadata`). Query-time behavior lives under **`retrieval`**.

### Methods

| `method` | Input | Behavior | Typical use |
|----------|-------|----------|-------------|
| `recursive` | Flat text or Markdown (`export_to_markdown()` from `DoclingDocument`) | Separator cascade (`\n\n` → `\n` → ` `) with `chunk_size` / `chunk_overlap` | General text, `speed` preset |
| `hybrid` | `DoclingDocument` tree | `HybridChunker`: structure-aware boundaries + tokenizer-aware split/merge; optional `include_metadata` / LLM contextual enrichment | Structured PDFs/DOCX, `balanced` preset |

[Docling chunking concepts](https://docling-project.github.io/docling/concepts/chunking/) distinguish **Markdown export + post-split** (recursive path) from **native chunkers on the document model** (hybrid).

### Extraction and optimization flow

```text
search_space_preparation → validate MaaS models, materialize chunking envelope
text_extraction → DoclingDocument (JSON/YAML + artifacts) → manifest
rag_templates_optimization → load DoclingDocument per trial → branch on chunking.method
  hybrid: HybridChunker → embed contextualized chunk when include_metadata / LLM enrichment
  recursive: export_to_markdown() → split string
```

One parse per document; trials branch on `chunking.method` without re-parsing PDFs. Persist with `doc.save_as_json()` / `save_as_yaml()` using `ImageRefMode.REFERENCED` and an `artifacts_dir`; record docling version in a sidecar manifest.

### Docling hybrid — `include_metadata`

Hybrid-only boolean on `pattern.json`. When `true`, indexing inlines structural (and, on `balanced`, LLM) context into the text sent to the embedding model, not as separate vector fields. The `pattern.json` field remains **`include_metadata`**.

| Field | Role |
|-------|------|
| `include_metadata` | Contextualize embed/BM25 text (`hybrid` only; explored on `balanced` preset) |

### Chunking parameters

| Parameter | Notes |
|-----------|-------|
| `method` | `recursive`, `hybrid` |
| `chunk_size`, `chunk_overlap` | Splitter limits (preset envelopes above) |
| `include_metadata` | Hybrid contextualization in embed text |

**Example (`balanced` hybrid trial):**

```json
{
  "chunking": {
    "method": "hybrid",
    "chunk_size": 1024,
    "chunk_overlap": 50,
    "include_metadata": true
  }
}
```

---

## Retrieval methods

| `search_mode` | Behavior |
|---------------|----------|
| `vector` | Embedding similarity only |
| `keyword` | BM25 only |
| `hybrid` | Vector + BM25 fused with RRF (recommended for production) |

| Parameter | Default | Role |
|-----------|---------|------|
| `method` | `simple` | Query-time retrieval strategy |
| `number_of_chunks` | `5` | Top-k chunks (typical range 3–20) |
| `search_mode` | `hybrid` | `vector`, `keyword`, or `hybrid` |
| `ranker_strategy` | — | `rrf` or `weighted` (hybrid) |
| `ranker_k` | — | RRF constant (typical 60) |
| `ranker_alpha` | — | Weighted fusion: 0 keyword ↔ 1 vector |
| `distance_metric` | `cosine` | Vector similarity metric |

**Example:**

```json
{
  "retrieval": {
    "method": "simple",
    "number_of_chunks": 5,
    "search_mode": "hybrid",
    "ranker_strategy": "rrf",
    "ranker_k": 60,
    "ranker_alpha": 0.5
  }
}
```

ai4rag explores chunking and retrieval combinations during optimization; GAM selects the best pattern by `optimization_metric`. Sampling respects `max_combinations` and product search-space rules.

---

## Related

- [RAG templates](./ODH-ADR-0003-rag-templates.md) — current simple template vs planned Graph RAG
- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md)
- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
- [Prompt tuning](./ODH-ADR-0006-prompt-tuning.md)
- [MaaS support](./ODH-ADR-0007-maas-support.md) — MaaS and vector-DB Connections
- [documents_rag_optimization_pipeline](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)
- [Docling chunking concepts](https://docling-project.github.io/docling/concepts/chunking/)
