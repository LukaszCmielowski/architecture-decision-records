# Open Data Hub - AutoRAG Optimization Settings

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

This ADR documents the AutoRAG documents RAG optimization pipeline public parameters, quality presets, and the chunking / retrieval search-space dimensions explored by ai4rag during GAM optimization.

## Why

Pipeline operators and Dashboard integrations need a stable contract for how optimization runs are configured (data sources, models, presets) and which chunking and retrieval knobs form the searchable configuration space. Capturing this as an ADR keeps the contract versioned alongside the AutoRAG architecture.

## Goals

* Define the public parameter surface of documents_rag_optimization_pipeline
* Document speed and balanced presets (resources, Docling behavior, chunking envelope)
* Specify chunking methods (recursive, hybrid) and retrieval modes (vector, keyword, hybrid) used in the search space

## Non-Goals

* RAG template composition beyond the current simple-RAG search space (see ODH-ADR-0003)
* pattern.json inference / indexing export schema (see ODH-ADR-0004)
* Evaluation metric backends and judge selection (see ODH-ADR-0005)

## How

Pipeline parameters, presets, and the **chunking / retrieval** search space for **`documents_rag_optimization_pipeline`** ([pipelines-components](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline)). ADR context: [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md).

## Table of contents

- [Pipeline parameters](#pipeline-parameters)
- [Presets](#presets)
- [Chunking methods](#chunking-methods)
- [Retrieval methods](#retrieval-methods)
- [Related](#related)

---

## Pipeline parameters

Public surface of [`pipeline.py`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py) (verify on your **pipelines-components** tag):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `test_data_secret_name` | `str` | (required) | Secret for S3-compatible test-data access (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`; `AWS_DEFAULT_REGION` optional) |
| `test_data_bucket_name` | `str` | (required) | Bucket containing the evaluation benchmark JSON |
| `test_data_key` | `str` | (required) | Object key of the test data file |
| `input_data_secret_name` | `str` | (required) | Secret for document corpus access (same key convention) |
| `input_data_bucket_name` | `str` | (required) | Bucket containing source documents |
| `input_data_key` | `str` | `""` | Object key or prefix for input documents |
| `ogx_secret_name` | `str` | (required) | OGX secret (`OGX_CLIENT_API_KEY`, `OGX_CLIENT_BASE_URL`) |
| `vector_io_provider_id` | `str` | (required) | Registered vector I/O provider id (e.g. Milvus) |
| `embedding_models` | `Optional[List[str]]` | `None` | Optional embedding model allow-list for the search space |
| `generation_models` | `Optional[List[str]]` | `None` | Optional generation model allow-list for the search space |
| `optimization_metric` | `str` | `overall_score` | GAM objective: `faithfulness`, `answer_correctness`, `context_correctness`, `answer_relevance`, `overall_score` |
| `optimization_max_rag_patterns` | `int` | `8` | Max patterns to evaluate and retain |
| `preset` | `str` | `speed` | Quality tier — maps to Docling extraction, chunking search space, and indexing defaults ([Presets](#presets)) |

---

## Presets

Each preset fixes **ingestion and chunking search-space defaults**; ai4rag still optimizes embedding, retrieval, and generation within that envelope.

| Preset | Resources (workload steps) | Chunking search space | Docling PDF tables | `query_rag` `max_threads` |
|--------|---------------------------|----------------------|--------------------|---------------------------|
| `speed` (default) | 4 vCPU / 16 GiB | `recursive` only | `do_table_structure: false` | 10 |
| `balanced` | 8 vCPU / 32 GiB | `recursive` + `hybrid` | `do_table_structure: true` ([TableFormer](https://docling-project.github.io/docling/guides/pdf-processing/)) | 4 |

**`balanced` additionally:** `include_metadata: true` on hybrid trials — embed text includes structural metadata via Docling `contextualize()` (headings, captions; no LLM). See [Chunking methods](#chunking-methods).

Docling extraction (`text_extraction`) is fixed per run by the preset, not repeated in each `pattern.json`. Use `speed` for plain text or quick runs; `balanced` for structured PDFs/DOCX with tables and headings.

---

## Chunking methods

Chunking splits documents into embeddable segments. **`chunking`** fields control boundaries and Docling serialization (including optional structural metadata via `include_metadata`). Query-time behavior lives under **`retrieval`**.

### Methods

| `method` | Input | Behavior | Typical use |
|----------|-------|----------|-------------|
| `recursive` | Flat text or Markdown (`export_to_markdown()` from `DoclingDocument`) | Separator cascade (`\n\n` → `\n` → ` `) with `chunk_size` / `chunk_overlap` | General text, `speed` preset |
| `hybrid` | `DoclingDocument` tree | `HybridChunker`: structure-aware boundaries + tokenizer-aware split/merge; optional `include_metadata` → Docling `contextualize()` | Structured PDFs/DOCX, `balanced` preset |

[Docling chunking concepts](https://docling-project.github.io/docling/concepts/chunking/) distinguish **Markdown export + post-split** (recursive path) from **native chunkers on the document model** (hybrid).

### Extraction and optimization flow

```text
text_extraction → DoclingDocument (JSON/YAML + artifacts) → manifest
rag_templates_optimization → load DoclingDocument per trial → branch on chunking.method
  hybrid: HybridChunker → embed contextualize(chunk) when include_metadata
  recursive: export_to_markdown() → split string
```

One parse per document; trials branch on `chunking.method` without re-parsing PDFs. Persist with `doc.save_as_json()` / `save_as_yaml()` using `ImageRefMode.REFERENCED` and an `artifacts_dir`; record docling version in a sidecar manifest.

### Docling hybrid — `include_metadata`

Hybrid-only boolean. When `true`, indexing uses Docling **`contextualize(chunk)`** — structural metadata (section headings, captions, etc.) is inlined into the text sent to the embedding model, not stored as separate vector fields. Implementation calls Docling’s API; the `pattern.json` field is **`include_metadata`**.

| Field | Role |
|-------|------|
| `include_metadata` | Use `contextualize(chunk)` for embed/BM25 text (`hybrid` only; explored on `balanced` preset) |

### Chunking parameters

| Parameter | Notes |
|-----------|-------|
| `method` | `recursive`, `hybrid` |
| `chunk_size`, `chunk_overlap` | Splitter limits |
| `include_metadata` | Docling hybrid — structural metadata in embed text (`contextualize()`) |

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
- [Docling chunking concepts](https://docling-project.github.io/docling/concepts/chunking/)
