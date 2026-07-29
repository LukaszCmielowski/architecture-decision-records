# Open Data Hub - AutoRAG MaaS Support

|                |            |
| -------------- | ---------- |
| Date           | 2026-07-29 |
| Scope          | AutoRAG Component |
| Status         | Proposed |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-79225](https://redhat.atlassian.net/browse/RHOAIENG-79225) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) |

## What

This ADR documents integrating **Model-as-a-Service (MaaS)** into AutoRAG for LLM discovery, chat completions, and judge invocations during hyperparameter optimization (HPO). Embeddings and vector storage use companion Connections until MaaS covers those capabilities natively.

## Why

MaaS is the RHOAI platform surface for listing and invoking chat-capable models (OpenAI-compatible routes, catalog, tenancy, Connections). AutoRAG HPO should:

* Discover generation and judge models from the **MaaS catalog** instead of a component-specific model registry
* Invoke chat and judge completions through **MaaS** with the same Connection patterns used elsewhere in RHOAI
* Keep the optimization search space aligned with models operators actually provision and govern via MaaS

Embeddings are not yet available through MaaS; a user-provided embedding Connection is the interim path. Vector upsert/search during trials uses LangChain adapters so retrieval HPO does not depend on a separate inference gateway.

## Goals

* Use MaaS for dynamic LLM discovery (chat and judge models)
* Use MaaS OpenAI-compatible routes for chat completions and judge scoring during HPO (and prompt pre-check when enabled)
* Accept a user-provided embedding model Connection until MaaS supports embeddings
* Use LangChain vector DB adapters for trial-time index and retrieval
* Preserve existing retrieval search-space fields (`search_mode`, `ranker_strategy`, `ranker_k`, `ranker_alpha`, `number_of_chunks`)
* Keep `pattern.json` as the authoritative pattern contract for indexing and inference consumers
* Derive an **AgentProfile** YAML artifact from each winning pattern for Gen AI Studio / Playground persistence

## Non-Goals

* Agentic RAG app deployment on OpenShift ([RHOAIENG-79226](https://redhat.atlassian.net/browse/RHOAIENG-79226))
* Native MaaS embedding support in this phase ([RHOAIENG-79228](https://redhat.atlassian.net/browse/RHOAIENG-79228) — follow-on to drop the user embedding Connection)
* PRAXIS / MCP retrieval tools (post-3.6)
* Agentic app codegen from patterns
* Changing evaluation metric definitions ([ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md))

## How

Wire **`documents_rag_optimization_pipeline`** to three Connections: MaaS (LLM discovery + chat/judge), a user embedding endpoint, and a LangChain-compatible vector DB. GAM, pattern emission, and metrics stay as defined in sibling ADRs.

## Table of contents

- [MaaS role in AutoRAG](#maas-role-in-autorag)
- [Pipeline Connections](#pipeline-connections)
- [HPO integration](#hpo-integration)
- [Trial execution](#trial-execution)
- [Retrieval search space](#retrieval-search-space)
- [Pattern artifacts](#pattern-artifacts)
  - [AgentProfile derivation](#agentprofile-derivation)
- [Follow-on](#follow-on)
- [Related](#related)

---

## MaaS role in AutoRAG

| Capability | Provider | Notes |
|------------|----------|-------|
| LLM discovery | **MaaS** | List available chat / judge models for the search space |
| Chat completion | **MaaS** | Generation during trials and production-oriented templates |
| Judge / LLM-as-judge | **MaaS** | Evaluation and optional prompt pre-check ([ODH-ADR-0006](./ODH-ADR-0006-prompt-tuning.md)) |
| Embeddings | User Connection | Interim — `uri` + `token` until [RHOAIENG-79228](https://redhat.atlassian.net/browse/RHOAIENG-79228) |
| Vector upsert / search | LangChain adapters | Milvus, pgvector, etc. via `vector_db_secret_name` |

Optional `generation_models` allow-lists constrain the **MaaS** model pool; embedding allow-lists constrain the configured embedding endpoint.

---

## Pipeline Connections

Public surface of `documents_rag_optimization_pipeline` (see also [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)):

| Parameter | Role |
|-----------|------|
| `maas_secret_name` | MaaS Connection — LLM discovery and chat / judge invocations |
| `embedding_model_secret_name` | User embedding Connection — properties `uri` and `token` |
| `vector_db_secret_name` | Vector DB Connection for LangChain adapters |

Data references (`test_data_*`, `input_data_*`) and optimization controls (`optimization_metric`, `optimization_max_rag_patterns`, `preset`) are unchanged in role. Model allow-lists apply to the MaaS catalog and the embedding Connection as above.

---

## HPO integration

1. **Discover** — Query MaaS for chat and judge models available to the namespace / project
2. **Complete** — Run generation through MaaS OpenAI-compatible routes
3. **Judge** — Score answers via MaaS when the evaluation path uses an LLM judge
4. **Embed** — Call the user embedding endpoint from `embedding_model_secret_name`
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

## Retrieval search space

Trial retrieval continues to explore the same dimensions documented in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md):

| Field | Values / role |
|-------|----------------|
| `search_mode` | `vector`, `keyword`, `hybrid` |
| `ranker_strategy` | `rrf` or `weighted` (hybrid) |
| `ranker_k` | RRF constant |
| `ranker_alpha` | Weighted fusion weight |
| `number_of_chunks` | Top-k chunk budget |

These map onto LangChain retriever / fusion configuration so published patterns stay comparable across runs.

---

## Pattern artifacts

| Artifact | Role |
|----------|------|
| `pattern.json` | **Source of truth** — settings, inference template, indexing spec, evaluation |
| `AgentProfile` YAML | **Derived** Gen AI Studio agent config for Playground persistence and Studio apply flows |

Inference consumers continue to use `inference.responses_template` as described in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md). Studio / Playground persistence uses the AgentProfile schema ([RHOAIENG-64608](https://redhat.atlassian.net/browse/RHOAIENG-64608); [schema proposal](https://gist.github.com/NickGagan/cd3028256ca7601e32160a72ddf1e7ca)).

### AgentProfile derivation

AutoRAG generates an AgentProfile document from `pattern.json` (+ Connection references). The profile is the Studio-facing snapshot of “what the agent is”; it does not replace `pattern.json` for indexing or HPO audit.

**Resource shape** (YAML under ConfigMap `data.profile.yaml` when persisted in-cluster):

| Field | Value |
|-------|--------|
| `apiVersion` | `genai.redhat.com/v1alpha1` |
| `kind` | `AgentProfile` |
| Storage (tentative) | ConfigMap labeled `opendatahub.io/agent-profile: "true"`, name `agent-profile-{uuid}` |

**`spec` mapping from AutoRAG pattern → AgentProfile:**

| AgentProfile field | Source from `pattern.json` / run context |
|--------------------|------------------------------------------|
| `displayName` | Pattern name (e.g. `pattern_01`) or user-facing label |
| `description` | Optional; AutoRAG run / pattern summary |
| `model.id` | `settings.generation.model_id` (MaaS catalog id) |
| `model.sourceType` | `maas` for HPO-selected MaaS models |
| `model.uri` | MaaS / OpenAI-compatible base URL from the MaaS Connection |
| `model.authorization` | MaaS: `maasSubscription` and/or `credentialsRef` to the MaaS Connection Secret — never inline secrets |
| `temperature` / `stream` / `maxOutputTokens` | From `inference.responses_template` when present; otherwise generation defaults |
| `prompt` | Optional MLflow (or equivalent) prompt ref when system text is registered; else omit and rely on `instructions` / Studio prompt workflow |
| `vectorStores.stores[]` | From `settings.vector_store_binding.vector_store_id` — ConfigMap `storeRef` (`gen-ai-aa-vector-stores` + key) or direct `id`, per Studio conventions |
| `vectorStores.maxNumResults` | `settings.retrieval.number_of_chunks` (aligned with `file_search.max_num_results`) |
| `mcpServers` / `guardrails` | Out of scope for AutoRAG HPO emission unless the pattern explicitly carries them (post-3.6 / separate work) |

**Minimal AutoRAG-derived example** (MaaS model + vector store):

```yaml
apiVersion: genai.redhat.com/v1alpha1
kind: AgentProfile
metadata:
  name: pattern-01-rag-agent
spec:
  displayName: pattern_01
  description: AutoRAG-optimized RAG pattern
  model:
    id: gpt-4.1-mini
    uri: https://maas.example.com/v1
    sourceType: maas
    authorization:
      maasSubscription: sub-example
  temperature: 0.0
  stream: false
  vectorStores:
    stores:
      - storeRef:
          kind: ConfigMap
          name: gen-ai-aa-vector-stores
          key: vs_coll_pattern_01
    maxNumResults: 5
```

**Studio flow:**

1. Pipeline stores `pattern.json` and derived AgentProfile YAML
2. User selects a pattern in Studio / Dashboard
3. Studio persists the AgentProfile (ConfigMap) and wires Playground / inference
4. Runtime retrieve-and-generate follows the pattern’s `responses_template` and vector store binding ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md))

Credentials stay out of the profile body: only Secret / subscription **references**, matching the AgentProfile security model.

---

## Follow-on

| Work | Ticket / note |
|------|----------------|
| Adopt MaaS embeddings; drop `embedding_model_secret_name` | [RHOAIENG-79228](https://redhat.atlassian.net/browse/RHOAIENG-79228) |
| Agentic RAG deployment from patterns | [RHOAIENG-79226](https://redhat.atlassian.net/browse/RHOAIENG-79226) |
| Align [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md) parameter table with MaaS Connections | Sibling ADR update |
| Point [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) / [ODH-ADR-0006](./ODH-ADR-0006-prompt-tuning.md) at MaaS for HPO consumers | Wording alignment |

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
