# Open Data Hub - AutoRAG MaaS Support

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-31 |
| Scope          | AutoRAG Component |
| Status         | Proposed |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-2440](https://redhat.atlassian.net/browse/RHAISTRAT-2440) · [RHOAIENG-79225](https://redhat.atlassian.net/browse/RHOAIENG-79225) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) |

## What

This ADR documents integrating **Model-as-a-Service (MaaS)** into AutoRAG for LLM discovery, embeddings, chat completions, Ragas evaluation (LLM and embeddings), and optional prompt pre-check during hyperparameter optimization (HPO). Vector storage uses LangChain adapters via a vector-DB Connection.

## Why

MaaS is the RHOAI platform surface for listing and invoking chat-capable models (OpenAI-compatible routes, catalog, tenancy, Connections). AutoRAG HPO should:

* Discover generation models from the **MaaS catalog** instead of a component-specific model registry
* Invoke chat completions, embeddings, and Ragas evaluation through **MaaS** with the same Connection patterns used elsewhere in RHOAI
* Keep the optimization search space aligned with models operators actually provision and govern via MaaS

## Goals

* Use MaaS for dynamic LLM discovery (generation models)
* Use MaaS OpenAI-compatible routes for embeddings, chat completions, and Ragas evaluation during HPO (and prompt pre-check when enabled)
* Use LangChain vector DB adapters for trial-time index and retrieval

## Non-Goals

* Agentic RAG app deployment on OpenShift ([RHOAIENG-79226](https://redhat.atlassian.net/browse/RHOAIENG-79226); [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation))
* Pattern artifact inventory, `pattern.json` schema, and GenAI Studio OGX test path ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md))
* Connection key catalogs ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#connections))
* Changing evaluation metric definitions ([ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md))

## How

Wire **`documents_rag_optimization_pipeline`** to MaaS (discovery, embeddings, chat, Ragas evaluation) and a LangChain-compatible vector DB. Optional Graph RAG adds `graph_db_secret_name`. GAM, pattern emission, and metrics stay as defined in sibling ADRs.

## Table of contents

- [MaaS role in AutoRAG](#maas-role-in-autorag)
- [Pipeline Connections](#pipeline-connections)
- [Trial execution](#trial-execution)
- [Follow-on](#follow-on)
- [Related](#related)

---

## MaaS role in AutoRAG

| Capability | Provider | Notes |
|------------|----------|-------|
| LLM discovery | **MaaS** | List available chat models for the search space |
| Chat completion | **MaaS** | Generation during trials |
| Ragas evaluation | **MaaS** | LLM and embedding calls for Ragas metrics ([ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md#ragas-runtime)) |
| Prompt pre-check | **MaaS** | Optional DSPy LLM-as-a-judge ([ODH-ADR-0006](./ODH-ADR-0006-prompt-tuning.md)) |
| Embeddings | **MaaS** | Embedding calls during trials |
| Vector upsert / search | LangChain adapters | Milvus, pgvector, etc. via `vector_db_secret_name` |

Optional `generation_models` allow-lists constrain the **MaaS** model pool; embedding allow-lists constrain the configured embedding endpoint.

---

## Pipeline Connections

Public surface of `documents_rag_optimization_pipeline`. **Secret key catalogs and backend selection:** [ODH-ADR-0002 — Connections](./ODH-ADR-0002-experiment-settings.md#connections).

| Parameter                | Role |
|--------------------------|------|
| `maas_secret_name`       | MaaS Connection — discovery, embeddings, chat, Ragas evaluation, optional prompt pre-check |
| `vector_db_secret_name` | Vector DB Connection for LangChain adapters |
| `graph_db_secret_name`  | Graph DB Connection for Graph RAG (Neo4j). Empty for simple RAG. |

Data references (`test_data_*`, `input_data_*`) and optimization controls (`optimization_metric`, `optimization_max_rag_patterns`, `preset`) are unchanged in role. Model allow-lists apply to the MaaS catalog.


---

## Trial execution

```text
chunks
  → embed (MaaS)
  → vector DB upsert (LangChain adapter)
  → LangChain search (vector / keyword / hybrid)
  → MaaS chat completion
  → evaluate metrics (Unitxt; Ragas LLM and embeddings via MaaS)
```

GAM selection, pattern emission, and metric backends remain as in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md), [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md), and [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md). Prompt pre-check ([ODH-ADR-0006](./ODH-ADR-0006-prompt-tuning.md)) shares the MaaS chat path when enabled.

---

## Follow-on

| Work | Ticket / note |
|------|----------------|
| Agentic RAG deployment from patterns | [RHOAIENG-79226](https://redhat.atlassian.net/browse/RHOAIENG-79226) — [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation) |
| Align pipeline Connections with MaaS / vector-DB / graph-DB | Done (2026-08-26) — [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#connections) |

---

## Related

- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) — parent AutoRAG architecture
- [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) — pipeline parameters and search space
- [ODH-ADR-0004-rag-pattern-inference](./ODH-ADR-0004-rag-pattern-inference.md) — `pattern.json`, retrieve / generate, Studio OGX test
- [ODH-ADR-0005-rag-pattern-evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md) — Unitxt and Ragas metrics; `optimization_metric` `{name, evaluator}`
- [ODH-ADR-0006-prompt-tuning](./ODH-ADR-0006-prompt-tuning.md) — DSPy pre-check (shares MaaS chat path)
- [RHOAIENG-79225](https://redhat.atlassian.net/browse/RHOAIENG-79225) — MaaS integration for AutoRAG HPO
- [RHOAIENG-79226](https://redhat.atlassian.net/browse/RHOAIENG-79226) — Agentic RAG deployment
