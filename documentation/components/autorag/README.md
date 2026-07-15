# AutoRAG

AutoRAG in Red Hat OpenShift AI automates document **Retrieval-Augmented Generation (RAG)** optimization with **Kubeflow Pipelines (Data Science Pipelines)**. Training logic is delivered as versioned **pipelines** and **reusable components** from the **pipelines-components** repository, orchestrating **ai4rag** for search-space exploration, GAM-based configuration selection, benchmark evaluation, and **RAG pattern** artifact emission.

For the formal architecture decision (goals, lifecycle, alternatives, roadmap), see the AutoRAG ADR: [ODH-ADR-0001-autorag](../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md).

## Source repository

Implementation assets live under:

- **Product fork:** [red-hat-data-services/pipelines-components](https://github.com/red-hat-data-services/pipelines-components) (fork of [opendatahub-io/pipelines-components](https://github.com/opendatahub-io/pipelines-components))

AutoRAG-specific paths:

| Area | Path in repository |
|------|---------------------|
| Optimization pipeline | [`pipelines/training/autorag/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag) |
| Indexing pipeline | [`pipelines/data_processing/autorag/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag) |
| Training components | [`components/training/autorag/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag) |
| Data processing components | [`components/data_processing/autorag/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag) |
| CI / image build (example) | [`Dockerfile.konflux.autorag`](https://github.com/red-hat-data-services/pipelines-components/blob/main/Dockerfile.konflux.autorag), Tekton under [`.tekton/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/.tekton) |

Upstream README for the overall repo layout (components vs pipelines, categories, contribution expectations): [pipelines-components README](https://github.com/red-hat-data-services/pipelines-components/blob/main/README.md).

## Lifecycle

AutoRAG spans three phases. Only optimization runs inside the managed optimization pipeline; indexing and inference are driven by per-pattern artifacts (`pattern.json`).

| Phase | Where it runs | Outcome |
|-------|---------------|---------|
| **1. Optimize** | `documents_rag_optimization_pipeline` | Ranked **RAG patterns** on a document sample + HTML leaderboard |
| **2. Index** | `documents_indexing_pipeline` (or pattern notebook) | Full corpus indexed into the pattern's vector store |
| **3. Infer** | Platform `POST /v1/responses` | Production Q&A using `inference.responses_template` from the pattern |

## Pipelines

| Pipeline | Purpose | Entry (Python package layout) |
|----------|---------|--------------------------------|
| **Documents RAG Optimization** | HPO over chunking, embedding, retrieval, and generation; emit `pattern.json`, notebooks, evaluation detail | [`pipelines/training/autorag/documents_rag_optimization_pipeline/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline) |
| **Documents Indexing** | Post-optimization: discover, extract, chunk, embed, and write the production vector index | [`pipelines/data_processing/autorag/documents_indexing_pipeline/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline) |

Declared **dependencies** in pipeline metadata include **Kubeflow Pipelines >= 2.15.2** and **ai4rag** (see each pipeline’s `metadata.yaml` and `README.md` for `lastVerified`, owners, and tags).

## Reusable components

### Data processing (`components/data_processing/autorag`)

| Component | Role |
|-----------|------|
| [test_data_loader](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag/test_data_loader) | Load benchmark JSON from S3-compatible storage for evaluation |
| [documents_discovery](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag/documents_discovery) | Discover source documents for optimization sample and indexing |
| [text_extraction](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag/text_extraction) | Structured extraction (Docling) into `DoclingDocument` artifacts |
| [documents_indexing](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag/documents_indexing) | Chunk, embed, and write vectors to the platform vector store |

### Training (`components/training/autorag`)

| Component | Role |
|-----------|------|
| [search_space_preparation](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag/search_space_preparation) | Model pre-selection, language detection, search-space report |
| [rag_templates_optimization](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag/rag_templates_optimization) | Run **ai4rag** HPO; emit per-pattern directories (`pattern.json`, `evaluation_results.json`, notebooks, leaderboard input) |
| [component_stage_map_publisher](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag/component_stage_map_publisher) | Publish component-to-stage map for Dashboard consumption |

Shared run-status templates and helpers live under [`components/training/autorag/shared/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag/shared).

## Runtime and storage model

- **Artifact store:** Pattern outputs land under **`rag_patterns/<pattern_subdir>/`** in the cluster artifact store (S3-compatible backend configured for Data Science Pipelines), plus a pipeline-wide HTML leaderboard.
- **S3 credentials:** Data load and discovery steps use namespace **Secrets** (for example `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`) aligned with **Connections** materialized for pipeline workloads.
- **Platform credentials:** OGX / Llama Stack access via `ogx_secret_name` (`OGX_CLIENT_API_KEY`, `OGX_CLIENT_BASE_URL`).
- **Vector store:** Optimization and indexing use the registered **vector I/O provider** (`vector_io_provider_id`); supported backends and retrieval modes are documented in feature docs and evolve without ADR changes.

## Pipeline parameters and search space

For pipeline inputs, presets, chunking methods, retrieval modes, and optimization controls:

- **[Experiment settings (pipeline parameters)](./features/experiment_settings.md)** — Public surface of `documents_rag_optimization_pipeline`, presets (`speed`, `balanced`), and the chunking / retrieval search space.

## Pattern artifacts

Each optimized pattern is consolidated in **`pattern.json`** — authoritative settings, inference template, indexing workflow spec, and evaluation summary.

| Topic | Documentation |
|-------|----------------|
| **`pattern.json` schema**, inference (`POST /v1/responses`), index building | [RAG pattern inference](./features/rag_pattern_inference.md) |
| **Metrics**, judge model, `evaluation_results.json` | [RAG pattern evaluation](./features/rag_pattern_evaluation.md) |

Per-pattern outputs also include **`indexing_notebook.ipynb`**, **`inference_notebook.ipynb`**, and **`evaluation_results.json`** (per-benchmark-row detail).

## Experiment tracking

When MLflow integration is enabled at the project level, AutoRAG follows the same KFP parent/child run model as AutoML (one child run per RAG pattern). Detailed MLflow mapping documentation is maintained separately on the feature-docs branch.

## Platform context

AutoRAG runs **on** the same Data Science Pipelines platform described in [Data Science Pipelines](../pipelines/README.md): DSPA stacks, Argo-based execution, and KFP v2-style pipeline definitions.

## Keeping this page accurate

When behavior, parameters, or artifact layouts change, update this document from the canonical **README.md** / **pipeline.py** / **metadata.yaml** files in **pipelines-components** and the [ai4rag](https://github.com/IBM/ai4rag) release you document.
