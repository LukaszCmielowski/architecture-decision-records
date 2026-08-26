# AutoRAG MLflow Integration

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-25 |
| Scope          | AutoRAG |
| Status         | Draft |
| Authors        | [Lukasz Cmielowski](@LukaszCmielowski) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | [AutoML MLflow Integration](../automl/ODH-ADR-0004-mlflow-integration.md) |

## What

Integrate MLflow experiment tracking, metrics logging, and per-benchmark tracing into the AutoRAG **Documents RAG optimization** Kubeflow pipeline from [pipelines-components](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline) on OpenShift AI Data Science Pipelines.

## Why

AutoRAG pipelines evaluate multiple RAG patterns (chunking, embedding, retrieval, generation configurations) but lack a unified way to compare runs side-by-side, trace individual benchmark requests through retrieval/generation/evaluation stages. MLflow integration provides experiment tracking, comparison UI, and per-request tracing using the same KFP + MLflow model already established by AutoML.

## Goals

* Align with AutoML's KFP + MLflow integration model: opt-in via `MLFLOW_TRACKING_URI`, parent/child run hierarchy, explicit API calls
* Log one nested child run per RAG pattern with params (chunking, embedding, retrieval, generation settings) and aggregate evaluation metrics
* Implement mandatory per-benchmark tracing when MLflow is enabled: one trace per benchmark request with `autorag.retrieval`, `autorag.generation`, and `autorag.evaluation` spans
* Log pointers to KFP artifacts (`pattern.json`, `evaluation_results.json`, notebooks) without copying data into MLflow

## Non-Goals

* Copying `evaluation_results.json` content into MLflow (KFP remains the source of truth)
* OTel Collector integration, `mlflow.genai.evaluate()` on traces, `log_expectation` / `log_feedback`, `mlflow.log_input()` on parent run
* MLflow Model Registry integration

## How

### KFP MLflow integration mode

AutoRAG uses the **same KFP + MLflow integration model** as AutoML. Components check whether `MLFLOW_TRACKING_URI` is set. If yes, they **resume the KFP parent run** and create **nested child runs** (one per RAG pattern), using explicit `mlflow.log_params()` / `mlflow.log_metrics()` / `mlflow.set_tags()` to log pattern metadata and pointers to KFP artifacts.

When MLflow integration is enabled at the project level, OpenShift AI injects the same environment variables into pipeline step pods as for AutoML (`MLFLOW_TRACKING_URI`, `MLFLOW_WORKSPACE`, `MLFLOW_EXPERIMENT_ID`, `MLFLOW_RUN_ID`, `MLFLOW_TRACKING_AUTH`).

### KFP artifacts produced by the pipeline

These paths represent the **RHOAI 3.5+** artifact structure and are the **source of truth** for what MLflow logging attaches to.

| Artifact / path role | Producing step | Layout and role |
|---------------------|----------------|-----------------|
| **Test data** | `test_data_loader` | Benchmark JSON on disk (input to search prep + optimization). |
| **Discovered documents** | `documents_discovery` | Descriptor of corpus objects for extraction. |
| **Extracted text** | `text_extraction` | Extracted document text (e.g. via **docling**) for ai4rag. |
| **`search_space_prep_report`** | `search_space_preparation` | YAML-serialized **search space** after phase-one validation. |
| **`rag_patterns`** (directory artifact) | `rag_templates_optimization` | One **subdirectory per pattern** (pattern name = folder name). Under each: **`pattern.json`**, **`evaluation_results.json`**, **`indexing_notebook.ipynb`**, **`inference_notebook.ipynb`**. |

**Per-pattern artifacts:**

| Artifact | Type | Purpose |
|----------|------|---------|
| **`pattern.json`** | JSON | Consolidated pattern record: `settings` (chunking, embedding, retrieval, generation, vector-store binding), `indexing.pipeline_spec`, `evaluation` metrics, `iteration` / `duration_seconds`. Single source of truth for registration, deployment, and code generation. |
| **`indexing_notebook.ipynb`** | Jupyter Notebook | Indexing notebook instantiated from templates, parameterized for this pattern. |
| **`inference_notebook.ipynb`** | Jupyter Notebook | Inference notebook instantiated from templates, parameterized for this pattern. |
| **`evaluation_results.json`** | JSON | Per-benchmark-row scores and I/O. |

### MLflow mapping model

One parent run per KFP pipeline execution; one nested child run per RAG pattern when MLflow tracking is enabled.

| MLflow concept | Proposed mapping |
|----------------|------------------|
| **Experiment** | Use KFP-managed experiment (`MLFLOW_EXPERIMENT_ID`). Otherwise: e.g. `autorag_documents_rag_optimization` plus optional suffix. |
| **Parent run** | Resume KFP parent (`MLFLOW_RUN_ID`). **Tags:** `kfp_run_id`, `kfp_run_name`, `pipeline_name`, dataset hashes or URIs (non-secret). **Params:** `preset`, `optimization_metric`, `optimization_max_rag_patterns`, `vector_db_secret_name`, `graph_db_secret_name` (when set), `image`, `kfp_version`, `ai4rag_version`. |
| **Child runs** | One nested child run per RAG pattern (folder name or `pattern.json` `name`). Enables side-by-side comparison of faithfulness / answer_correctness / context_correctness and chunking / retrieval / model choices. |
| **Traces** | **Required** when `MLFLOW_TRACKING_URI` is set. One trace per benchmark request, attached to the pattern child run. P patterns x N benchmark rows = P x N traces total. |
| **Spans** | **Required** under each trace: `autorag.retrieval`, `autorag.generation`, `autorag.evaluation` with MLflow `SpanType` where applicable. Generation may include nested spans from `mlflow.openai.autolog()` for MaaS chat completions. |
| **Params (child)** | From `pattern.json` `settings`: `chunking.*`, `embedding.model_id`, `retrieval.*`, `generation.model_id` / prompt fields, `vector_store_binding` (`provider_type`, `collection_name`). |
| **Metrics (child)** | From `pattern.json`: `duration_seconds`, `iteration`; from `evaluation.metrics[]`: `log_metric(name, scores.mean)` per entry; objective score from the entry with `optimization_metric: true` (logged as `final_score`). |
| **Child Artifacts** | Pointers to KFP artifacts: `pattern.json`, `evaluation_results.json`, `indexing_notebook.ipynb`, `inference_notebook.ipynb`. Logged as params containing artifact URIs/paths, not copied into MLflow. |

### Implementation approach

All logging is driven from `rag_templates_optimization` using each pattern's `pattern.json`. No separate MLflow work in `leaderboard_evaluation` — the HTML leaderboard and MLflow UI both reflect the same pattern-level scores.

| Step | Where | Action |
|------|-------|--------|
| 1 | `rag_templates_optimization` | If `MLFLOW_TRACKING_URI` is unset, skip all MLflow code (lazy-import `mlflow`). |
| 2 | Same | Resume parent run: `mlflow.start_run(run_id=os.environ["MLFLOW_RUN_ID"])`. Log parent params/tags. |
| 3 | Same, before optimization | `mlflow.tracing.enable()` and `mlflow.openai.autolog()`. |
| 4 | Per pattern (in ai4rag or component) | Open a nested child run; for each benchmark row create one trace with `autorag.*` spans and `SpanType`s; then `log_params` / `log_metrics` / KFP pointers from `pattern.json`. |

**Parent run discovery:** KFP-injected `MLFLOW_RUN_ID` and `MLFLOW_EXPERIMENT_ID` are sufficient. A separate MLflow tracking artifact is not required.

### Metrics logged (child runs)

ai4rag computes RAGAS-style metrics during optimization. Log **aggregates only** on the pattern child run:

| Source field | MLflow |
|--------------|--------|
| Objective metric `scores.mean` | `log_metric("final_score", ...)` from the `metrics[]` entry with `optimization_metric: true` |
| `metrics[].name` | `log_metric(name, scores.mean)` for each entry in `evaluation.metrics` |
| `duration_seconds` | `log_metric("duration_seconds", ...)` |

Per-question rows stay in KFP `evaluation_results.json`.

### Tracing per pattern child run

When `MLFLOW_TRACKING_URI` is set, traces and stage spans are **mandatory**, scoped to the pattern child run.

**Trace and run hierarchy:**

| Level | Scope | Count (example: 3 patterns, 10 benchmark rows) |
|-------|-------|--------------------------------------------------|
| **Parent run** | One KFP pipeline execution | 1 |
| **Child run** | One RAG pattern | 3 |
| **Trace** | One eval request (one benchmark row) | 10 per child run, 30 total |

```
Parent run (pipeline)
└── Child run: pattern_A
    ├── Trace: autorag.pattern_A.query_0
    │   ├── autorag.retrieval      (SpanType.RETRIEVER)
    │   ├── autorag.generation     (SpanType.CHAT_MODEL)
    │   └── autorag.evaluation
    ├── Trace: autorag.pattern_A.query_1
    │   └── …
    └── … (N traces = benchmark rows)
```

**Invariant:** For pattern P, `count(traces under child_run(P)) == count(benchmark eval requests for P)`.

**Spans and span types:**

| Span name | `SpanType` | Inputs / outputs (summary) |
|-----------|------------|----------------------------|
| `autorag.retrieval` | `RETRIEVER` | Query in; retrieved documents out (`page_content`, `metadata.doc_uri`, `metadata.chunk_id`) per MLflow retriever schema |
| `autorag.generation` | `CHAT_MODEL` | Query + context in; answer out; `mlflow.chat.tokenUsage`; `ai.model.name` / `ai.model.provider` |
| `autorag.evaluation` | (default) | Ground truth, prediction, context in; per-metric scores out; `metric.{name}` attributes |

Enable `mlflow.openai.autolog()` at component start so MaaS OpenAI-compatible chat calls inside `autorag.generation` produce nested spans without duplicating the full request body.

**Dependencies:** `mlflow>=2.22` (OpenAI chat autolog); `openai` in the component or ai4rag image.

**Verification:** In the MLflow UI: parent run > child run > **Traces** (N rows) > expand one trace to see `autorag.retrieval`, `autorag.generation`, `autorag.evaluation`.

## Alternatives

### 1. Attach traces to the parent run instead of pattern child runs

**Discarded:** Mixing traces from all patterns under the parent run loses the per-pattern grouping. With P patterns x N benchmark rows, traces become difficult to filter and compare. The child-run-per-pattern hierarchy preserves the natural comparison unit and aligns with AutoML's one-child-run-per-model approach.

### 2. Use a single trace per pattern (all benchmark rows in one trace)

**Discarded:** This violates the one-trace-per-request convention and produces unwieldy traces with hundreds of spans. One trace per benchmark row keeps traces focused and allows the MLflow UI to display latency/token breakdowns per individual query.

### 3. Copy evaluation_results.json content into MLflow

**Discarded:** KFP artifacts are the source of truth for per-row evaluation data. Duplicating this into MLflow adds storage overhead and creates sync issues. MLflow logs aggregate metrics and pointers to the KFP artifacts instead.

## Risks

* **Trace volume at scale** — P patterns x N benchmark rows can produce large numbers of traces (e.g., 50 patterns x 100 rows = 5,000 traces per pipeline run). *Mitigation: Monitor MLflow server capacity; consider sampling for large benchmark sets in future releases.*
* **`mlflow>=2.22` requirement** — OpenAI chat autolog requires a recent MLflow version that may not be available in all RHOAI deployments. *Mitigation: Lazy-import and version-check; fall back to manual generation span logging if autolog is unavailable.*

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| AutoRAG team                  |                  |            | Yes |
| Data Science Pipelines        |                  |            | Yes |
| MLflow integration (RHOAI)    | Matt Prahl, Humair Khan |  | Yes |
| ai4rag library                |                  |            | Yes |

## References

* AutoML MLflow Integration ADR: [ODH-ADR-0004-mlflow-integration](../automl/ODH-ADR-0004-mlflow-integration.md)
* AutoRAG ADR: [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
* AutoRAG optimization settings: [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md)
* RAG pattern evaluation: [ODH-ADR-0005-rag-pattern-evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
* MLflow Tracking: [MLflow Tracking documentation](https://mlflow.org/docs/latest/tracking.html)
* MLflow span types: [MLflow span types documentation](https://mlflow.org/docs/latest/genai/concepts/span#span-types)
* MLflow retriever schema: [MLflow retriever spans](https://mlflow.org/docs/latest/genai/concepts/span#retriever-spans)
* OpenAI tracing: [MLflow OpenAI tracing](https://mlflow.org/docs/latest/genai/tracing/integrations/listing/openai/)
* Upstream pipeline: [pipelines-components — documents_rag_optimization_pipeline](https://github.com/opendatahub-io/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
