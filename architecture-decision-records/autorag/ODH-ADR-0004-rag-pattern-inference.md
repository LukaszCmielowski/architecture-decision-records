# Open Data Hub - AutoRAG Pattern Inference

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-26 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1846](https://redhat.atlassian.net/browse/RHAISTRAT-1846) · [RHAISTRAT-1731](https://redhat.atlassian.net/browse/RHAISTRAT-1731) · [RHAISTRAT-1724](https://redhat.atlassian.net/browse/RHAISTRAT-1724) · [RHAISTRAT-1424](https://redhat.atlassian.net/browse/RHAISTRAT-1424) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) · [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0007-maas-support](./ODH-ADR-0007-maas-support.md) |

## What

This ADR documents AutoRAG pattern artifacts after optimization: the pattern.json schema, production retrieve-and-generate via MaaS (OpenAI-compatible chat completions) plus the LangChain vector store, and full-corpus index building through the managed documents indexing pipeline.

## Why

Optimized configurations must be portable across optimization, indexing, and inference. A durable pattern contract lets Dashboard and APIs select a winning pattern, rebuild the production index, and reconstruct retrieve-and-generate from `settings` (the same chunking, retrieval, and generation fields used during benchmarking). There is no frozen request-body template in `pattern.json`.

## Goals

* Define the target pattern.json schema (`settings`, `indexing`, `evaluation`; no `inference` block)
* Document how `settings.generation` and `settings.retrieval` drive MaaS chat completions and LangChain retrieval
* Document GenAI Studio / Playground retrieve-and-generate wiring from pattern artifacts
* Document indexing.pipeline_spec for the managed documents-indexing-pipeline

## Non-Goals

* Metric computation and judge calibration details (see ODH-ADR-0005)
* Graph RAG pattern storage profiles beyond sketches in ODH-ADR-0003
* OpenShift-native operator deployment of agents (future work referenced in ODH-ADR-0003)

## How

This page describes **AutoRAG patterns** after optimization: the **`pattern.json`** schema, **retrieve and generation** via **MaaS** and the pattern's vector-store binding, and **index building** into the production vector store.

## Table of contents

- [Optimization pipeline](#optimization-pipeline)
- [pattern.json schema](#patternjson-schema)
  - [Target schema](#target-schema)
- [Example pattern.json](#example-patternjson)
- [Retrieve and generation](#retrieve-and-generation)
  - [GenAI Studio flow](#genai-studio-flow)
- [Index building](#index-building)
- [Related](#related)

---

## Optimization pipeline

The **[`documents_rag_optimization_pipeline`](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)** runs **`rag_templates_optimization`** to search RAG configurations and score each candidate on a benchmark (up to 1 GB document sample). Outputs land under **`rag_patterns/<pattern_subdir>/`** in DSPA storage (`<bucket>/<pipeline-name>/<run-id>/…`) plus a pipeline-wide HTML leaderboard.

Each **`pattern.json`** captures optimized **`settings`**, **`indexing`** (pipeline spec), and **`evaluation`** results. Index building processes the **full document corpus** into the vector store the pattern queries at inference time. Consumers assemble chat completions from `settings.generation`; they do **not** read an `inference` / `responses_template` object.

| Artifact | Purpose |
|----------|---------|
| `pattern.json` | Authoritative record: `name`, `settings`, `indexing`, `evaluation`, `iteration`, `max_combinations`, `duration_seconds` |
| `agentic_template.zip` | Parameterized [agentic RAG starter-kit](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) for this pattern (Studio / deploy) |
| `indexing_notebook.ipynb`, `inference_notebook.ipynb` | Parameterized notebooks for the pattern |
| `evaluation_results.json` | Per-question detail ([`evaluation_results.json`](./ODH-ADR-0005-rag-pattern-evaluation.md#evaluation_resultsjson)) |

---

## pattern.json schema

### Target schema

```text
pattern.json
├── name, iteration, max_combinations, duration_seconds
├── settings
│   ├── vector_store_binding (provider_type, collection_name)
│   ├── chunking (method, chunk_size, chunk_overlap, include_metadata)
│   ├── embedding (model_id, embedding_params)
│   ├── retrieval (method, number_of_chunks, search_mode, ranker_strategy, ranker_alpha)
│   └── generation (model_id, temperature, max_completion_tokens,
│                   context_template_text, user_message_text,
│                   system_message_text, language)
├── indexing
│   └── pipeline_spec
│       ├── pipeline_name
│       ├── parameters
│       └── overrides_allowed
└── evaluation
    └── metrics[]
        ├── evaluator, name, description, scores (mean, ci_low, ci_high)
        ├── model_id (judge entries only)
        └── optimization_metric: true (exactly one entry — GAM objective)
```

| Field | Description |
|-------|-------------|
| `name`, `iteration`, `max_combinations`, `duration_seconds` | Pattern identity, GAM iteration, search-space size, wall time |
| `settings` | Optimized RAG config: `vector_store_binding` (`provider_type`, `collection_name`), `chunking` (incl. `include_metadata`), `embedding`, `retrieval` (`method`, `number_of_chunks`, `search_mode`, ranker fields), `generation` (model, sampling, `context_template_text` / `user_message_text` / `system_message_text`, `language` `{code, name}`) |
| `indexing.pipeline_spec` | Managed indexing pipeline inputs — [Index building](#index-building) |
| `evaluation` | `metrics[]` — per-metric `evaluator`, `name`, `description`, `scores` (`mean`, `ci_low`, `ci_high`); `model_id` on judge entries; exactly one entry has `optimization_metric: true` (GAM objective). See [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md) |

`pattern.json` has **no** `inference` object and **no** `responses_template`. Prompt text lives only under `settings.generation`.

GAM ranks patterns by the pipeline [`optimization_metric`](./ODH-ADR-0002-experiment-settings.md) parameter (default `overall_score`). The matching `evaluation.metrics[]` entry is marked `optimization_metric: true`; its `scores.mean` is the pattern objective score.

---

## Example pattern.json

```json
{
  "name": "Pattern1",
  "max_combinations": 90,
  "evaluation": {
    "metrics": [
      {
        "name": "answer_correctness",
        "evaluator": "unitxt",
        "description": "Measures how accurately the generated answer matches the ground-truth reference answers.",
        "scores": {
          "mean": 0.7161,
          "ci_low": 0.6071,
          "ci_high": 0.8149
        }
      },
      {
        "name": "faithfulness",
        "evaluator": "unitxt",
        "description": "Measures whether the generated answer is grounded in the retrieved context without hallucination.",
        "scores": {
          "mean": 0.9063,
          "ci_low": 0.8806,
          "ci_high": 0.9292
        }
      },
      {
        "name": "context_correctness",
        "evaluator": "unitxt",
        "description": "Measures whether the retrieved context passages match the ground-truth reference documents.",
        "scores": {
          "mean": 0.95,
          "ci_low": 0.8,
          "ci_high": 1.0
        }
      },
      {
        "name": "answer_relevance",
        "evaluator": "judge",
        "description": "LLM judge score for how directly and helpfully the response addresses the question.",
        "scores": {
          "mean": 0.95,
          "ci_low": 0.85,
          "ci_high": 1.0
        },
        "model_id": "publishers/ai-eng-cracow/models/qwen3-8b-fp8-dynamic"
      },
      {
        "name": "overall_score",
        "evaluator": "custom",
        "description": "Aggregate score computed as the mean of all other evaluated metrics.",
        "scores": {
          "mean": 0.8806,
          "ci_low": 0.7844,
          "ci_high": 0.936
        },
        "optimization_metric": true
      }
    ]
  },
  "duration_seconds": 48,
  "settings": {
    "vector_store_binding": {
      "provider_type": "milvus",
      "collection_name": "ai4rag_20260824161307_t2pxa4s3"
    },
    "chunking": {
      "method": "hybrid",
      "chunk_size": 512,
      "chunk_overlap": 0,
      "include_metadata": true
    },
    "embedding": {
      "model_id": "publishers/ai-eng-cracow/models/redhataibge-m3",
      "embedding_params": {
        "embedding_dimension": 1024,
        "context_length": 1015
      }
    },
    "retrieval": {
      "method": "simple",
      "number_of_chunks": 5,
      "search_mode": "hybrid",
      "ranker_strategy": "weighted",
      "ranker_alpha": 0.5
    },
    "generation": {
      "model_id": "publishers/ai-eng-cracow/models/qwen3-8b-fp8-dynamic",
      "temperature": 0.2,
      "max_completion_tokens": 2048,
      "context_template_text": "Document {doc_number}:\n{document}",
      "user_message_text": "\nContext:\n{reference_documents}\n\nQuestion: {question}\nRespond exclusively in English, regardless of any other language used in the provided context. You MUST respond in English.",
      "system_message_text": "Please answer the user's question based solely on the provided context documents. If the question cannot be answered from the provided context, say you cannot answer. Your answer should be concise.",
      "language": {
        "code": "en",
        "name": "English"
      }
    }
  },
  "iteration": 0,
  "indexing": {
    "pipeline_spec": {
      "pipeline_name": "documents-indexing-pipeline",
      "parameters": {
        "maas_secret_name": "maas",
        "vector_db_secret_name": "milvus",
        "input_data_secret_name": "minio",
        "input_data_bucket_name": "jwalaszc-bucket",
        "input_data_key": "rh_summit_2026/documents",
        "batch_size": 20,
        "provider_type": "milvus",
        "collection_name": "ai4rag_20260824161307_t2pxa4s3",
        "embedding_model_id": "publishers/ai-eng-cracow/models/redhataibge-m3",
        "embedding_params": {
          "embedding_dimension": 1024,
          "context_length": 1015
        },
        "chunking_method": "hybrid",
        "chunk_size": 512,
        "chunk_overlap": 0
      },
      "overrides_allowed": [
        "input_data_secret_name",
        "input_data_bucket_name",
        "input_data_key",
        "collection_name",
        "batch_size"
      ]
    }
  }
}
```

---

## Retrieve and generation

Optimization and production generation both use **MaaS OpenAI-compatible chat completions** (`MAAS_BASE_URL`, `MAAS_API_KEY`) — `POST {MAAS_BASE_URL}/v1/chat/completions`. Retrieval uses the LangChain vector-store adapter for `settings.vector_store_binding.provider_type` / `collection_name` (Milvus or PGVector credentials from `vector_db_secret_name`). Query-time knobs come from `settings.retrieval` (`method`, `number_of_chunks`, `search_mode`, `ranker_strategy`, `ranker_alpha`).

Assemble the chat request from **`settings.generation`**:

| Field | Role |
|-------|------|
| `model_id` | Chat `model` |
| `temperature`, `max_completion_tokens` | Sampling |
| `system_message_text` | System message |
| `context_template_text` | Per-chunk formatting (`{doc_number}`, `{document}`) |
| `user_message_text` | User message (`{reference_documents}`, `{question}`) |

**Python (generation body after retrieval):**

```python
import json, os
from pathlib import Path
import requests

def format_context(pattern: dict, documents: list[str]) -> str:
    tmpl = pattern["settings"]["generation"]["context_template_text"]
    return "\n\n".join(
        tmpl.format(doc_number=i, document=doc)
        for i, doc in enumerate(documents, start=1)
    )

def chat_completions_body(pattern: dict, question: str, documents: list[str]) -> dict:
    gen = pattern["settings"]["generation"]
    user = gen["user_message_text"].format(
        reference_documents=format_context(pattern, documents),
        question=question,
    )
    return {
        "model": gen["model_id"],
        "temperature": gen["temperature"],
        "max_completion_tokens": gen["max_completion_tokens"],
        "messages": [
            {"role": "system", "content": gen["system_message_text"]},
            {"role": "user", "content": user},
        ],
    }

def generate(pattern_path: Path, question: str, documents: list[str]) -> dict:
    pattern = json.loads(pattern_path.read_text())
    base = os.environ["MAAS_BASE_URL"].rstrip("/")
    r = requests.post(
        f"{base}/v1/chat/completions",
        json=chat_completions_body(pattern, question, documents),
        headers={"Authorization": f"Bearer {os.environ['MAAS_API_KEY']}"},
        timeout=120,
    )
    r.raise_for_status()
    return r.json()
```

Retrieve chunks first (LangChain adapter + `settings.retrieval` / `settings.vector_store_binding`), then pass chunk texts into `generate`.

### GenAI Studio flow

Studio / Playground persistence uses the AgentProfile schema ([RHOAIENG-64608](https://redhat.atlassian.net/browse/RHOAIENG-64608); [schema proposal](https://gist.github.com/NickGagan/cd3028256ca7601e32160a72ddf1e7ca)). `agentic_template.zip` is the deployable starter-kit bundle for that pattern.

1. Pipeline stores `pattern.json` and `agentic_template.zip` under the pattern subdirectory
2. User selects a pattern in Studio / Dashboard
3. Studio persists the AgentProfile (ConfigMap) and wires Playground / inference from `pattern.json` `settings` (and optionally unpacks `agentic_template.zip`)
4. Runtime retrieve-and-generate follows the pattern’s `settings` (generation templates + vector-store binding)

---

## Index building

Index building populates the production vector store via the managed **`documents-indexing-pipeline`** ([`documents_indexing_pipeline`](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/data_processing/autorag/documents_indexing_pipeline/pipeline.py)), registered in the AI Pipelines catalog. One pipeline definition serves all patterns; per-pattern values come from **`indexing.pipeline_spec`**.

| `pipeline_spec` field | Role |
|-----------------------|------|
| `pipeline_name` | Managed catalog name (e.g. `documents-indexing-pipeline`) |
| `parameters` | Pre-filled from optimization run + pattern `settings` |
| `overrides_allowed` | Keys the UI may expose for user override at submit time |

**Parameter sources:** optimization run → `maas_secret_name`, `vector_db_secret_name`, `input_data_*`; pattern `settings` → embedding (`embedding_model_id`, `embedding_params`), chunking, `collection_name` / `provider_type`. Secret fields are **names only** (Kubernetes Secret references).

**Workflow:** optimization completes → user selects pattern → read `pipeline_spec` → resolve managed pipeline → pre-fill run form → user confirms/overrides → submit → full corpus indexed → [retrieve and generation](#retrieve-and-generation) ready.

**Pipeline steps:** load inputs → document discovery/extraction → chunking → embedding (MaaS) → vector store write → validation/logging. Observable via KFP; re-runnable when documents or overrides change.

---

## Related

- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
- [RAG templates](./ODH-ADR-0003-rag-templates.md) — current simple template vs planned Graph RAG
- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md) — pipeline parameters, presets, chunking, retrieval
- [MaaS support](./ODH-ADR-0007-maas-support.md) — MaaS and vector-DB Connections
- [RHOAIENG-64608](https://redhat.atlassian.net/browse/RHOAIENG-64608) — AgentProfile schema for Gen AI Studio
- [AgentProfile schema proposal](https://gist.github.com/NickGagan/cd3028256ca7601e32160a72ddf1e7ca) — ConfigMap / `profile.yaml` example and field definitions
- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
