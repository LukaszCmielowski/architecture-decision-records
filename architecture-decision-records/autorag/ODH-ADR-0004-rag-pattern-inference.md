# Open Data Hub - AutoRAG Pattern Inference

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-31 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1846](https://redhat.atlassian.net/browse/RHAISTRAT-1846) · [RHAISTRAT-1731](https://redhat.atlassian.net/browse/RHAISTRAT-1731) · [RHAISTRAT-1724](https://redhat.atlassian.net/browse/RHAISTRAT-1724) · [RHAISTRAT-1424](https://redhat.atlassian.net/browse/RHAISTRAT-1424) · [RHAISTRAT-2623](https://redhat.atlassian.net/browse/RHAISTRAT-2623) · [RHOAIENG-88692](https://redhat.atlassian.net/browse/RHOAIENG-88692) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) · [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) |

## What

This ADR documents AutoRAG pattern artifacts after optimization: the pattern.json schema, production retrieve-and-generate via MaaS (OpenAI-compatible chat completions) plus the LangChain vector store, three consumers of that contract (inference notebook, starter-kit zip with Helm, one-click Agent Sandbox), GenAI Studio as a separate OGX test backend, and full-corpus index building through the managed documents indexing pipeline.

## Why

Optimized configurations must be portable across optimization, indexing, and inference. A durable pattern contract lets Dashboard and APIs select a winning pattern, rebuild the production index, and reconstruct retrieve-and-generate from `settings` (the same chunking, retrieval, and generation fields used during benchmarking). There is no frozen request-body template in `pattern.json`.

## Goals

* Define the target pattern.json schema (`settings`, `indexing`, `evaluation`; no `inference` block)
* Document how `settings.generation` and `settings.retrieval` drive MaaS chat completions and LangChain retrieval
* Document the inference notebook, parameterized starter-kit zip, Helm / BuildConfig deploy, and one-click Agent Sandbox
* Document GenAI Studio as an OGX-based test path (not the agent `/chat/completions` contract)
* Document indexing.pipeline_spec for the managed documents-indexing-pipeline

## Non-Goals

* Metric catalog and score computation (see ODH-ADR-0005)
* Graph RAG pattern storage profiles beyond sketches in ODH-ADR-0003

## How

This page describes **AutoRAG patterns** after optimization: the **`pattern.json`** schema, **retrieve and generation** via **MaaS** and the pattern's vector-store binding (notebook, starter-kit zip with Helm, one-click Agent Sandbox), **GenAI Studio** as a separate **OGX** test backend, and **index building** into the production vector store.

## Table of contents

- [Optimization pipeline](#optimization-pipeline)
- [Pattern artifacts](#pattern-artifacts)
- [pattern.json](#patternjson)
  - [Example pattern.json](#example-patternjson)
- [Retrieve and generation](#retrieve-and-generation)
  - [Inference notebook](#inference-notebook)
  - [Agentic Starter-kit](#agentic-starter-kit)
  - [One-click Deployment](#one-click-deployment)
  - [GenAI Studio integration](#genai-studio-integration)
- [Index building](#index-building)
- [Related](#related)

---

## Optimization pipeline

The **[`documents_rag_optimization_pipeline`](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)** runs **`rag_templates_optimization`** to search RAG configurations and score each candidate on a benchmark (up to 1 GB document sample). Outputs land under **`rag_patterns/<pattern_subdir>/`** in DSPA storage (`<bucket>/<pipeline-name>/<run-id>/…`) plus a pipeline-wide HTML leaderboard.

Each **`pattern.json`** captures optimized **`settings`**, **`indexing`** (pipeline spec), and **`evaluation`** results. The same `run_rag_optimization()` call in **ai4rag** writes notebooks and **`starter_kit.zip`** into that directory (see [Agentic Starter-kit](#agentic-starter-kit)). Index building processes the **full document corpus** (union of `input_data_keys` locations) into the vector store the pattern queries at inference time. Consumers assemble chat completions from `settings.generation`; they do **not** read an `inference` / `responses_template` object.

---

## Pattern artifacts

Canonical per-pattern inventory. Sibling ADRs link here instead of repeating this table. Row-level score schema: [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md#evaluation_resultsjson). Zip Helm and one-click serving: [Retrieve and generation](#retrieve-and-generation). MLflow pointers: [ODH-ADR-0006](./ODH-ADR-0006-mlflow-integration.md).

| Artifact | Purpose |
|----------|---------|
| `pattern.json` | Authoritative record: `name`, `settings`, `indexing`, `evaluation`, `iteration`, `max_combinations`, `duration_seconds` |
| `starter_kit.zip` | Parameterized [agentic RAG starter-kit](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) for IDE work and Helm (`make deploy`). Built by ai4rag from **importable package resources** (same mechanism as the notebooks). One-click serving uses the prebuilt `autorag-inference` image, not this zip. |
| `indexing_notebook.ipynb`, `inference_notebook.ipynb` | Parameterized notebooks: full-corpus index vs retrieve-and-generate with a sample query |
| `evaluation_results.json` | Per-question detail ([`evaluation_results.json`](./ODH-ADR-0005-rag-pattern-evaluation.md#evaluation_resultsjson)) |

---

## pattern.json

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
        ├── evaluator (unitxt | ragas | custom), name, description, scores (mean, ci_low, ci_high)
        ├── model_id (ragas entries when recorded)
        └── optimization_metric: true (exactly one entry — GAM objective {name, evaluator})
```

| Field | Description |
|-------|-------------|
| `name`, `iteration`, `max_combinations`, `duration_seconds` | Pattern identity, GAM iteration, search-space size, wall time |
| `settings` | Optimized RAG config: `vector_store_binding` (`provider_type`, `collection_name`), `chunking` (incl. `include_metadata`), `embedding`, `retrieval` (`method`, `number_of_chunks`, `search_mode`, ranker fields), `generation` (model, sampling, `context_template_text` / `user_message_text` / `system_message_text`, `language` `{code, name}`) |
| `indexing.pipeline_spec` | Managed indexing pipeline inputs — [Index building](#index-building) |
| `evaluation` | `metrics[]` aggregates. Catalog, evaluators, and GAM flag: [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md) |

`pattern.json` has **no** `inference` object and **no** `responses_template`. Prompt text lives only under `settings.generation`.

GAM ranks patterns by the pipeline [`optimization_metric`](./ODH-ADR-0002-experiment-settings.md) parameter. Semantics, default, and allowed `{name, evaluator}` pairs: [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md#optimization_metric). The matching `evaluation.metrics[]` entry is marked `optimization_metric: true`; its `scores.mean` is the pattern objective score.

---

### Example pattern.json

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
        "name": "faithfulness",
        "evaluator": "ragas",
        "description": "Ragas faithfulness: whether the generated answer is grounded in retrieved context.",
        "scores": {
          "mean": 0.874,
          "ci_low": 0.82,
          "ci_high": 0.92
        },
        "model_id": "publishers/ai-eng-cracow/models/qwen3-8b-fp8-dynamic"
      },
      {
        "name": "answer_relevancy",
        "evaluator": "ragas",
        "description": "Ragas answer relevancy: how on-topic the answer is versus the question.",
        "scores": {
          "mean": 0.912,
          "ci_low": 0.86,
          "ci_high": 0.95
        },
        "model_id": "publishers/ai-eng-cracow/models/redhataibge-m3"
      },
      {
        "name": "context_precision",
        "evaluator": "ragas",
        "description": "Ragas context precision: whether relevant retrieved contexts are ranked high.",
        "scores": {
          "mean": 0.868,
          "ci_low": 0.80,
          "ci_high": 0.93
        },
        "model_id": "publishers/ai-eng-cracow/models/qwen3-8b-fp8-dynamic"
      },
      {
        "name": "context_recall",
        "evaluator": "ragas",
        "description": "Ragas context recall: how much of the ground-truth answer is present in retrieved context.",
        "scores": {
          "mean": 0.938,
          "ci_low": 0.88,
          "ci_high": 0.98
        },
        "model_id": "publishers/ai-eng-cracow/models/qwen3-8b-fp8-dynamic"
      },
      {
        "name": "overall_score",
        "evaluator": "custom",
        "description": "Equal-weight mean of every other metric that ran (Unitxt and Ragas on the product path).",
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
        "input_data_keys": [
          "rh_summit_2026/documents",
          "rh_summit_2026/policies"
        ],
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
        "input_data_keys",
        "collection_name",
        "batch_size"
      ]
    }
  }
}
```

---

## Retrieve and generation

Optimization and production both use **MaaS** for chat completions (`POST {MAAS_BASE_URL}/v1/chat/completions`) and for **embeddings** (indexing and query-time vector search). Retrieval uses the LangChain vector-store adapter for `settings.vector_store_binding.provider_type` / `collection_name` (Milvus or PGVector credentials from `vector_db_secret_name`). Query-time knobs come from `settings.retrieval` (`method`, `number_of_chunks`, `search_mode`, `ranker_strategy`, `ranker_alpha`).

Assemble the chat request from **`settings.generation`**:

| Field | Role |
|-------|------|
| `model_id` | Chat `model` |
| `temperature`, `max_completion_tokens` | Sampling |
| `system_message_text` | System message |
| `context_template_text` | Per-chunk formatting (`{doc_number}`, `{document}`) |
| `user_message_text` | User message (`{reference_documents}`, `{question}`) |

Retrieve chunks first (LangChain adapter + `settings.retrieval` / `settings.vector_store_binding`), then pass chunk texts into generation. Three consumers share that contract, all parameterized from the same `pattern.json` `settings`. The full-corpus index must exist first ([Index building](#index-building)). [GenAI Studio](#genai-studio-integration) is documented after those paths: it is a separate **OGX** test backend, not a fourth caller of the agent API.

| Path | Audience | Intent |
|------|----------|--------|
| [Inference notebook](#inference-notebook) | Data scientist | Inspect retrieve → generate in Jupyter with a sample query |
| [Agentic Starter-kit](#agentic-starter-kit) | Developer | Download zip, customize code, `make run-app` / Helm `make deploy` |
| [One-click Deployment](#one-click-deployment) | Operator / Dashboard | One-click default RAG: prebuilt `autorag-inference` image on [Agent Sandbox](https://agent-sandbox.sigs.k8s.io/docs/) |

### Inference notebook

`inference_notebook.ipynb` sits next to `pattern.json` under `rag_patterns/<pattern_name>/`. It is for **inspecting** retrieve-and-generate in the workbench. It is not a server: it does not start FastAPI, Helm, or a sandbox. Distinct from `indexing_notebook.ipynb`, which builds the full-corpus index.

The notebook is instantiated from a template and pre-filled from `pattern.json`. Typical cells:

1. Load `pattern.json` and the MaaS / vector-DB Connections (`MAAS_BASE_URL`, `MAAS_API_KEY`, `vector_db_secret_name`). Secrets stay in Connections; they are not inlined in the notebook.
2. Bind the LangChain vector store from `settings.vector_store_binding` (`provider_type`, `collection_name`).
3. Run a **sample query** (a placeholder, or a row from the optimization benchmark, that the user can replace).
4. Retrieve top-k chunks using `settings.retrieval` (`number_of_chunks`, `search_mode`, ranker fields). Query embeddings use MaaS (`settings.embedding.model_id`).
5. Format context with `settings.generation.context_template_text` (`{doc_number}`, `{document}`).
6. Call MaaS `POST /v1/chat/completions` with `system_message_text`, `user_message_text` (`{reference_documents}`, `{question}`), `model_id`, `temperature`, and `max_completion_tokens`.
7. Display the answer and retrieved chunks.

### Agentic Starter-kit

`starter_kit.zip` is the **downloadable, editable** agent: LangGraph orchestration in `src/agentic_rag`, FastAPI in `main.py` (`POST /chat/completions`, `GET /health`), in-app chat UI at `GET /` (`playground/`), plus `Makefile`, `Dockerfile`, Helm chart, and `.env.example`. That `GET /` UI is local to the kit; it is not GenAI Studio.

**Generation.** ai4rag owns the kit as **package resources** (an importable file tree with placeholders), the same way it owns notebook templates. `run_rag_optimization()` copies that tree, fills placeholders from the pattern's `settings`, and writes **`starter_kit.zip`** next to `pattern.json`. There is no unpublished zip of the base kit. Layout and prompts stay aligned with the [agentic RAG starter-kit](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag); ai4rag is the copy used at optimize time so filled settings match the zip. `rag_templates_optimization` stays a thin wrapper: no extra KFP component.

| Concern | Detail |
|---------|--------|
| **Orchestration** | LangGraph: route query → retrieve → generate → respond |
| **API** | `POST /chat/completions` (streaming / non-streaming), `GET /health` |
| **Local run** | `make run-app` (uvicorn on `PORT`, default 8000) or `make run-cli` |
| **Cluster deploy** | Helm chart → OpenShift (`make deploy`) or [one-click](#one-click-deployment) with the prebuilt image |
| **Observability** | Optional MLflow tracing (`MLFLOW_TRACKING_URI`) — [ODH-ADR-0006](./ODH-ADR-0006-mlflow-integration.md) |

**User flow**

1. Download `starter_kit.zip` from the pattern subdirectory.
2. Open it in an IDE (workbench or local).
3. **Setup:** `make init` (`.env` from `.env.example`; AutoRAG already writes pattern values), `make env` (`uv`, Python 3.12).
4. Point `BASE_URL` / `API_KEY` at the project MaaS Connection (`BASE_URL` must end with `/v1`). The production collection already exists; do **not** run the kit's sample `make load-docs` / `data/load_documents.py` against it (those load `DOCS_TO_LOAD` into a new local collection).
5. **Tweak:** edit `src/agentic_rag` (graph, tools, prompts), `main.py`, or `.env`.
6. **Run locally:** `make run-app` or `make run-cli`.
7. **Deploy a custom image:** Helm (`make build` or `make build-openshift`, then `make deploy`). Default (unedited) app: [One-click Deployment](#one-click-deployment).

### One-click Deployment

**One-click** serving of the **default** RAG application: no zip unpack, no user image build. Dashboard (or API) starts a pod from the prebuilt **`autorag-inference`** image with env taken from `pattern.json` `settings` and project Connections. The image exposes the same contract as the kit: `POST /chat/completions`, `GET /health`.

Runtime is [Agent Sandbox](https://agent-sandbox.sigs.k8s.io/docs/) (SIG Apps): a Kubernetes-native singleton with a stable hostname, optional persistent storage, hibernation (pause idle compute; resume on network activity), and scheduled deletion (TTL). Isolation is a cluster choice (standard containers, gVisor, or Kata) via `RuntimeClass`; AutoRAG does not ship a custom operator for this path.

| Object | API | AutoRAG use |
|--------|-----|-------------|
| [`SandboxTemplate`](https://agent-sandbox.sigs.k8s.io/docs/getting_started/overview/) | `extensions.agents.x-k8s.io` | Reusable pod spec: `autorag-inference` image, ports, security context. `envVarsInjectionPolicy` must be `Allowed` or `Overrides` so a claim can set pattern env. |
| `SandboxWarmPool` (optional) | `extensions.agents.x-k8s.io` | Pre-warms pods from the template for fast **adoption** when the claim sets **no** extra env. |
| `SandboxClaim` | `extensions.agents.x-k8s.io` | Dashboard/user request. Injects pattern env via `spec.env` (same map as the zip `.env`). |
| `Sandbox` | `agents.x-k8s.io` | Bound instance. Stable hostname for `/chat/completions` callers. |

**Env vs warm pool:** Agent Sandbox bakes env into the Pod at create time. A `SandboxClaim` that sets `spec.env` **cannot adopt** a warm-pool pod; the controller cold-starts from the template ([API: `SandboxClaim.spec.env`](https://agent-sandbox.sigs.k8s.io/docs/api/)). AutoRAG MVP accepts that: pattern settings differ per claim, so one-click is “prebuilt image + cold start from template,” still cheaper than `make build` / Helm. A later design can keep a generic image in the warm pool and supply `pattern.json` after start (no per-claim `spec.env`) if millisecond adopt is required.

**Flow:** user selects a pattern (index already built) → Dashboard creates a `SandboxClaim` against the AutoRAG inference template → controller creates a `Sandbox` from the template with claim env + MaaS / vector-DB secrets → `GET /health` then `POST /chat/completions`. Python and Go [clients](https://agent-sandbox.sigs.k8s.io/docs/getting_started/overview/) can claim the same template without the UI.

This is the default-app path. Edited starter-kit code uses Helm (`make deploy`). The kit's `make deploy-openshell` / `Containerfile.openshell` is a related isolated-runtime experiment; product one-click is Agent Sandbox + `autorag-inference`.

### GenAI Studio integration

GenAI Studio is **not** a consumer of the agent contract (`POST /chat/completions` on `autorag-inference` or the starter-kit). It is an interactive **test** path: AutoRAG Dashboard materializes an AgentProfile from `pattern.json`, and Studio runs retrieve-and-generate on its **OGX** backend.

OGX is a different stack from MaaS + LangChain (notebook) and from LangGraph FastAPI (zip / one-click). Answers can differ from those paths even with the same `pattern.json`. That is expected.

Persistence uses the AgentProfile schema ([RHOAIENG-64608](https://redhat.atlassian.net/browse/RHOAIENG-64608); [schema proposal](https://gist.github.com/NickGagan/cd3028256ca7601e32160a72ddf1e7ca)). Dashboard copies **vector-store, retrieval, and generation** from `pattern.json` into that profile:

| `pattern.json` | AgentProfile |
|----------------|--------------|
| `settings.vector_store_binding` (`provider_type`, `collection_name`) | Vector-store binding |
| `indexing.pipeline_spec.parameters.vector_db_secret_name` | Vector-DB Connection **name** (secret reference; credentials stay on the Connection) |
| `settings.retrieval` (`method`, `number_of_chunks`, `search_mode`, `ranker_strategy`, `ranker_alpha`) | Retrieval |
| `settings.generation` (`model_id`, `temperature`, `max_completion_tokens`, `system_message_text`, `user_message_text`, `context_template_text`, `language`) | Generation |

1. Pipeline stores `pattern.json` under the pattern subdirectory.
2. User selects a pattern and (if needed) runs [index building](#index-building).
3. AutoRAG Dashboard persists an AgentProfile (ConfigMap) from the fields above.
4. The AgentProfile is opened in Studio and wired to Studio's OGX backend.

---

## Index building

Index building populates the production vector store via the managed **`documents-indexing-pipeline`** ([`documents_indexing_pipeline`](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/data_processing/autorag/documents_indexing_pipeline/pipeline.py)), registered in the AI Pipelines catalog. One pipeline definition serves all patterns; per-pattern values come from **`indexing.pipeline_spec`**.

| `pipeline_spec` field | Role |
|-----------------------|------|
| `pipeline_name` | Managed catalog name (e.g. `documents-indexing-pipeline`) |
| `parameters` | Pre-filled from optimization run + pattern `settings` |
| `overrides_allowed` | Keys the UI may expose for user override at submit time |

**Parameter sources:** optimization run → `maas_secret_name`, `vector_db_secret_name`, `input_data_*`; pattern `settings` → embedding (`embedding_model_id`, `embedding_params`), chunking, `collection_name` / `provider_type`. Secret fields are **names only** (Kubernetes Secret references).

`parameters.input_data_keys` is the same **`list[str]`** as the optimization run (1–10 object keys or prefixes, same Connection/bucket). Production indexing uses that full list so the production corpus matches optimization. A one-element list is a single location. Corpus contract: [ODH-ADR-0002 — Corpus locations](./ODH-ADR-0002-experiment-settings.md#corpus-locations).

**Workflow:** optimization completes → user selects pattern → read `pipeline_spec` → resolve managed pipeline → pre-fill run form → user confirms/overrides → submit → full corpus indexed → [retrieve and generation](#retrieve-and-generation) ready.

**Pipeline steps:** load inputs → document discovery/extraction → chunking → embedding (MaaS) → vector store write → validation/logging. Observable via KFP; re-runnable when documents or overrides change.

---

## Related

- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
- [RAG templates](./ODH-ADR-0003-rag-templates.md) — simple vs Neo4j Graph RAG templates; which kit path each maps to
- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md) — pipeline parameters, corpus list, benchmark JSON, presets, chunking, retrieval, HPO MaaS / vector Connections
- [MLflow integration](./ODH-ADR-0006-mlflow-integration.md) — tracking pointers to pattern artifacts
- [RHOAIENG-64608](https://redhat.atlassian.net/browse/RHOAIENG-64608) — AgentProfile schema for Gen AI Studio
- [AgentProfile schema proposal](https://gist.github.com/NickGagan/cd3028256ca7601e32160a72ddf1e7ca) — ConfigMap / `profile.yaml` example and field definitions
- [OGX](https://ogx-ai.github.io/docs) — Studio test backend (not the one-click / zip agent)
- [Agentic RAG starter-kit](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) — zip generation template
- [Agent Sandbox](https://agent-sandbox.sigs.k8s.io/docs/) — one-click `Sandbox` / `SandboxTemplate` / `SandboxClaim` / `SandboxWarmPool`
- [Agent Sandbox API](https://agent-sandbox.sigs.k8s.io/docs/api/) — `SandboxClaim.spec.env` cold-start behavior
- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
