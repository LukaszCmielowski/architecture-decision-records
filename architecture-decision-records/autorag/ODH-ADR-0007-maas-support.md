# Open Data Hub - AutoRAG MaaS Support

|                |            |
| -------------- | ---------- |
| Date           | 2026-07-29 |
| Scope          | AutoRAG Component |
| Status         | Proposed |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-2440](https://redhat.atlassian.net/browse/RHAISTRAT-2440) · [RHOAIENG-79225](https://redhat.atlassian.net/browse/RHOAIENG-79225) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) |

## What

This ADR documents integrating **Model-as-a-Service (MaaS)** into AutoRAG for LLM discovery, chat completions, and judge invocations during hyperparameter optimization (HPO). Embeddings and vector storage use companion Connections until MaaS covers those capabilities natively.

## Why

MaaS is the RHOAI platform surface for listing and invoking chat-capable models (OpenAI-compatible routes, catalog, tenancy, Connections). AutoRAG HPO should:

* Discover generation and judge models from the **MaaS catalog** instead of a component-specific model registry
* Invoke chat and judge completions through **MaaS** with the same Connection patterns used elsewhere in RHOAI
* Keep the optimization search space aligned with models operators actually provision and govern via MaaS

## Goals

* Use MaaS for dynamic LLM discovery (chat and judge models)
* Use MaaS OpenAI-compatible routes for chat completions and judge scoring during HPO (and prompt pre-check when enabled)
* Accept a user-provided embedding model Connection until MaaS supports embeddings
* Use LangChain vector DB adapters for trial-time index and retrieval
* Preserve existing retrieval search-space fields (`search_mode`, `ranker_strategy`, `ranker_k`, `ranker_alpha`, `number_of_chunks`)
* Keep `pattern.json` as the authoritative pattern contract for indexing and inference consumers

## Non-Goals

* Agentic RAG app deployment on OpenShift ([RHOAIENG-79226](https://redhat.atlassian.net/browse/RHOAIENG-79226))
* Agentic app codegen from patterns
* Changing evaluation metric definitions ([ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md))

## How

Wire **`documents_rag_optimization_pipeline`** to three Connections: MaaS (LLM discovery + chat/judge), and a LangChain-compatible vector DB. GAM, pattern emission, and metrics stay as defined in sibling ADRs.

## Table of contents

- [MaaS role in AutoRAG](#maas-role-in-autorag)
- [Pipeline Connections](#pipeline-connections)
- [HPO integration](#hpo-integration)
- [Trial execution](#trial-execution)
- [Pattern artifacts](#pattern-artifacts)
- [Follow-on](#follow-on)
- [Related](#related)

---

## MaaS role in AutoRAG

| Capability | Provider | Notes |
|------------|----------|-------|
| LLM discovery | **MaaS** | List available chat / judge models for the search space |
| Chat completion | **MaaS** | Generation during trials and production-oriented templates |
| Judge / LLM-as-judge | **MaaS** | Evaluation and optional prompt pre-check ([ODH-ADR-0006](./ODH-ADR-0006-prompt-tuning.md)) |
| Embeddings |  **MaaS** | Generation during trials and production-oriented templates |
| Vector upsert / search | LangChain adapters | Milvus, pgvector, etc. via `vector_db_secret_name` |

Optional `generation_models` allow-lists constrain the **MaaS** model pool; embedding allow-lists constrain the configured embedding endpoint.

---

## Pipeline Connections

Public surface of `documents_rag_optimization_pipeline` (see also [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)):

| Parameter                | Role |
|--------------------------|------|
| `maas_secret_name`       | MaaS Connection — LLM discovery and chat / judge invocations |
| `vector_db_secret_name` | Vector DB Connection for LangChain adapters |

Data references (`test_data_*`, `input_data_*`) and optimization controls (`optimization_metric`, `optimization_max_rag_patterns`, `preset`) are unchanged in role. Model allow-lists apply to the MaaS catalog.

**Milvus**
```json
{
    "MILVUS_URI": "MILVUS_URI",
    "MILVUS_TOKEN": "MILVUS_TOKEN",
    "MILVUS_SERVER_CERT": "MILVUS_SERVER_CERT",
}
```

**PGVector**
```json
{
  "PGVECTOR_HOST": "PGVECTOR_HOST",
  "PGVECTOR_PORT": "PGVECTOR_PORT",
  "PGVECTOR_DB": "PGVECTOR_DB",
  "PGVECTOR_USER": "PGVECTOR_USER",
  "PGVECTOR_PASSWORD": "PGVECTOR_PASSWORD"
}
```


---

## HPO integration

1. **Discover** — Query MaaS for chat and judge models available to the namespace / project
2. **Complete** — Run generation through MaaS OpenAI-compatible routes
3. **Judge** — Score answers via MaaS when the evaluation path uses an LLM judge
4. **Embed** — Call the embedding endpoint through MaaS OpenAI-compatible routes
5. **Retrieve** — Upsert and search via LangChain adapters from `vector_db_secret_name`

ai4rag talks to MaaS through the platform Connection; it does not require a separate foundation-model client for HPO chat/judge traffic.

---

## Trial execution

```text
chunks
  → embed (user embedding endpoint)
  → vector DB upsert (LangChain adapter)
  → LangChain search (vector / keyword / hybrid)
  → MaaS chat completion
  → evaluate metrics (judge via MaaS when applicable)
```

GAM selection, `pattern.json` emission, and metric backends remain as in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md), [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md), and [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md). Prompt pre-check ([ODH-ADR-0006](./ODH-ADR-0006-prompt-tuning.md)) shares the MaaS chat/judge path when enabled.

---

## Pattern artifacts

| Artifact                    | Role                                                                          |
|-----------------------------|-------------------------------------------------------------------------------|
| `pattern.json`              | **Source of truth** — settings, inference template, indexing spec, evaluation |
| `agentic_template.zip` | agentic starter kit aligned with https://github.com/red-hat-data-services/agentic-starter-kits |

Inference consumers continue to use `inference.responses_template` as described in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md). Studio / Playground persistence uses the AgentProfile schema ([RHOAIENG-64608](https://redhat.atlassian.net/browse/RHOAIENG-64608); [schema proposal](https://gist.github.com/NickGagan/cd3028256ca7601e32160a72ddf1e7ca)).

### GenAI Studio flow

1. Pipeline stores `pattern.json` 
2. User selects a pattern in Studio / Dashboard
3. Studio persists the AgentProfile (ConfigMap) and wires Playground / inference
4. Runtime retrieve-and-generate follows the pattern’s `responses_template` and vector store binding ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md))

---

## Follow-on

| Work | Ticket / note |
|------|----------------|
| Adopt MaaS embeddings; drop `embedding_model_secret_name` | [RHOAIENG-79228](https://redhat.atlassian.net/browse/RHOAIENG-79228) |
| Agentic RAG deployment from patterns | [RHOAIENG-79226](https://redhat.atlassian.net/browse/RHOAIENG-79226) |
| Align [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md) / [0003](./ODH-ADR-0003-rag-templates.md) / [0004](./ODH-ADR-0004-rag-pattern-inference.md) / [0006](./ODH-ADR-0006-prompt-tuning.md) with MaaS Connections | Done (2026-08-26) |

---

## Related

- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) — parent AutoRAG architecture
- [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) — pipeline parameters and search space
- [ODH-ADR-0004-rag-pattern-inference](./ODH-ADR-0004-rag-pattern-inference.md) — `pattern.json` and inference template
- [ODH-ADR-0005-rag-pattern-evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md) — metrics and judge
- [ODH-ADR-0006-prompt-tuning](./ODH-ADR-0006-prompt-tuning.md) — DSPy pre-check (shares MaaS chat/judge path)
- [RHOAIENG-79225](https://redhat.atlassian.net/browse/RHOAIENG-79225) — MaaS integration for AutoRAG HPO
- [RHOAIENG-79228](https://redhat.atlassian.net/browse/RHOAIENG-79228) — MaaS embeddings adoption
- [RHOAIENG-79226](https://redhat.atlassian.net/browse/RHOAIENG-79226) — Agentic RAG deployment
- [RHOAIENG-64608](https://redhat.atlassian.net/browse/RHOAIENG-64608) — AgentProfile schema for Gen AI Studio
- [AgentProfile schema proposal](https://gist.github.com/NickGagan/cd3028256ca7601e32160a72ddf1e7ca) — ConfigMap / `profile.yaml` example and field definitions
