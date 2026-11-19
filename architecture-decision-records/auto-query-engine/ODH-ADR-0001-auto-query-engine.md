# Open Data Hub - Auto Query Engine (Structured RAG over Flight SQL and DCH)

|                |            |
| -------------- | ---------- |
| Date           | 2026-11-19 |
| Scope          | AutoRAG Component |
| Status         | Proposed |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | TBD |
| Other docs:    | [ODH-ADR-0002](../autorag/ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0004](../autorag/ODH-ADR-0004-rag-pattern-inference.md) · [ODH-ADR-0005](../autorag/ODH-ADR-0005-rag-pattern-evaluation.md) · [ODH-ADR-0006](../autorag/ODH-ADR-0006-mlflow-integration.md) · [ODH-ADR-0008](../autorag/ODH-ADR-0008-openai-compatible-model-support.md) |

## What

Extend AutoRAG with an **Auto Query Engine** for automated Text-to-SQL / Structured RAG optimization against structured data sources exposed through **Data Connect Hub (DCH)** and executed via **Apache Arrow Flight SQL**. It shares experiment orchestration, model bindings, tracking, and artifact management, while providing a dedicated Text-to-SQL workflow and reusable inference module.

## Why

Enterprise knowledge is split between unstructured documents (handled by document AutoRAG) and structured tables in data warehouses, lakehouses, and operational databases. Vector-based RAG fails on aggregations, multi-table joins, temporal comparisons, and exact numerical lookups.

At the same time, raw LLM Text-to-SQL accuracy degrades sharply on real enterprise schemas due to large table counts, ambiguous column names, dialect differences, and domain-specific categorical values. By applying AutoRAG's automated hyperparameter/pattern search, offline indexing, and deterministic evaluation architecture on top of **DCH** (connection governance) and **Arrow Flight SQL** (standardized, high-throughput SQL execution), OpenShift AI users can optimize and deploy governed Structured RAG pipelines without custom connector engineering or external SaaS dependencies.

## Non-Goals

- Database provisioning, ETL/ELT ingestion, or schema migration management.
- Write/mutation workloads (`INSERT`, `UPDATE`, `DELETE`, `DDL`); Auto Query Engine is strictly read-only.
- Replacing DCH connection management or building bespoke per-database JDBC/ODBC drivers outside Arrow Flight SQL.
- Automatic multi-source cross-database federated joins unless natively supported by the underlying Flight SQL endpoint (e.g., Trino/Starburst/Dremio).

## How

### Architecture Overview

Users select Document RAG or Structured data / Text-to-SQL when creating an AutoRAG experiment. Both workflows share experiment settings, trial budgets and ranking, model bindings ([ODH-ADR-0008](../autorag/ODH-ADR-0008-openai-compatible-model-support.md)), KFP v2 orchestration, MLflow tracking ([ODH-ADR-0006](../autorag/ODH-ADR-0006-mlflow-integration.md)), and results and artifact management.

The structured-query implementation is a separate module with dedicated pipeline inputs, search dimensions, and evaluation metrics. It owns schema profiling, SQL generation and validation, dialect handling, and DCH / Flight SQL execution:

1. **Schema Profiling & Semantic Indexing (Offline Phase):** Connects to the target DCH data source via Arrow Flight SQL (`GetCatalogs`, `GetDbSchemas`, `GetTables`, `GetPrimaryKeys`), collects bounded column statistics and distinct categorical value samples, and constructs a searchable schema representation.
2. **Pattern Search & Optimization (KFP Trials):** Evaluates candidate Text-to-SQL pipeline configurations across schema-linking, semantic enrichment, few-shot retrieval, prompting, self-correction, and LLM choices against a benchmark dataset.
3. **Winning Pattern Export:** Persists the top-ranked query engine pattern (schema index, prompt templates, few-shot store, dialect rules, model binding, and DCH connection reference) for downstream inference via notebooks, Llama Stack / MCP tools, or deployed inference services.

### Pipeline Inputs

| Parameter | Type | Requirement | Description |
| --------- | ---- | ----------- | ----------- |
| `dch_connection_name` | `str` | Required | Name of the DCH connection resource in the run's namespace defining the Flight SQL endpoint, catalog/schema scope, and auth configuration. |
| `target_tables` | `list[str]` | Optional | Allowlist of `schema.table` patterns to scope profiling and query generation. Defaults to all accessible tables in the DCH connection scope. |
| `benchmark_dataset_ref` | `str` | Optional | Reference to evaluation dataset of `(question, golden_sql)` or `(question, expected_result)` pairs. If omitted, triggers synthetic benchmark generation with optional human-in-the-loop review. |
| `model_bindings` | — | Required | Mutually exclusive MaaS (`maas_secret_name`, `generation_models`, `embedding_models`) or direct OpenAI-compatible Secrets (`generation_model_secret_name`, `embedding_model_secret_name`) per [ODH-ADR-0008](../autorag/ODH-ADR-0008-openai-compatible-model-support.md). |
| `max_query_timeout_sec` | `int` | Optional | Hard execution timeout per Flight SQL query during profiling and evaluation trials (default: `30`). |
| `max_rows_limit` | `int` | Optional | Automatic `LIMIT` ceiling injected into unaggregated queries during evaluation and inference (default: `1000`). |

### Search Space Dimensions

Each optimization trial composes a candidate Structured RAG pipeline across the following dimensions:

| Dimension | Candidate Strategies |
| --------- | -------------------- |
| **Schema Linking & Pruning** | Full DDL injection (small schemas), vector similarity over table/column embeddings, foreign-key 1-hop graph expansion. |
| **Semantic Enrichment** | Raw DDL only, DDL + sample distinct values (`SELECT DISTINCT`), LLM-enriched column/table descriptions cached during profiling. |
| **Few-Shot / Golden SQL Retrieval** | Zero-shot, static top-$k$ examples, dynamic similarity retrieval over validated `(question, SQL)` pairs. |
| **Generation & Prompt Strategy** | Direct SQL generation, Chain-of-Thought decomposition, CTE-first modular query synthesis, dialect-specific system instructions (ANSI, Trino, Postgres, DuckDB). |
| **Validation & Self-Correction** | Single-pass execution vs. Flight SQL `Prepare` / `EXPLAIN` syntax & schema check with $N \in \{1, 2\}$ error-reflection retries. |
| **Model Selection** | Candidate generation and embedding models supplied via MaaS or direct OpenAI-compatible bindings. |

### Evaluation Metrics and Ranking

The workflow reuses AutoRAG's evaluation orchestration and ranking, with dedicated SQL metrics rather than document-RAG retrieval and answer metrics. [ODH-ADR-0005](../autorag/ODH-ADR-0005-rag-pattern-evaluation.md) must be extended to describe this evaluation contract:

- **Execution Accuracy (EX) [Primary]:** Percentage of evaluation questions whose generated SQL returns an Arrow result set semantically equivalent to the golden SQL result (order-invariant row/column comparison with configurable floating-point tolerance).
- **Valid SQL Rate:** Percentage of generated queries that pass AST read-only validation and Flight SQL `Prepare`/`EXPLAIN` without syntax, missing-column, or permission errors.
- **End-to-End Latency & Token Cost:** Combined LLM generation time, retry overhead, and Flight SQL execution duration.

Trials failing safety validation or exceeding timeout thresholds are penalized or disqualified from winning-pattern selection.

### Cold-Start Synthetic Benchmark Generation

When users lack a curated `(question, golden_sql)` evaluation dataset:

1. A profiling task samples valid join paths, aggregations, filters, and temporal queries directly from the DCH schema metadata.
2. Candidate SQL queries are executed via Flight SQL to confirm syntactic validity and non-empty result sets.
3. The generation model back-translates verified SQL queries into natural-language questions, producing a reviewable synthetic benchmark artifact stored alongside the experiment in MLflow.

### Winning Pattern Artifact and Consumption

Winning patterns use a distinct `structured_query` pattern type within AutoRAG's artifact lifecycle. [ODH-ADR-0004](../autorag/ODH-ADR-0004-rag-pattern-inference.md) must be extended with its inference contract and consumer dispatch before existing consumers can load this type. Structured-query patterns record:

- DCH connection reference (`dch_connection_name`), catalog/schema scope, and SQL dialect.
- Serialized schema index, semantic enrichments, and few-shot retrieval store.
- Prompt templates, retry policy, and SQL safety guardrails (`max_rows_limit`, `max_query_timeout_sec`).
- Model binding metadata (`maas` or `openai`, model IDs, endpoints, Secret names).

At inference time, the reusable query module loads the winning pattern, generates SQL, validates it via AST inspection, executes it over Arrow Flight SQL using the runtime caller's DCH authorization context, and optionally synthesizes a natural-language summary grounded in the returned Arrow table. Notebooks, MCP tools, and deployed applications can consume this module independently of the KFP optimization pipeline.

## Alternatives

| Option | Advantages | Drawbacks |
| ------ | ---------- | --------- |
| AutoRAG structured-query workflow on DCH + Arrow Flight SQL (selected) | Shares experiment orchestration, model bindings, tracking, and results UX; reuses DCH connection governance and standardized SQL execution. | Requires Flight SQL endpoint availability, dedicated SQL evaluation, and a structured-query artifact contract. |
| Auto Query Engine as a DCH capability | Places natural-language querying alongside connection discovery, authorization, and Flight SQL access. | Extends DCH's data-access scope into model selection, optimization trials, evaluation, and inference; requires experiment orchestration and results UX. |
| Auto Query Engine as a Training Service workload | Could reuse the emerging service's job submission, scheduling, and backend lifecycle capabilities. | Requires a structured-query optimization runtime and an API beyond the current training-specific model/dataset/algorithm contract; query inference and results UX still need ownership. |
| Standalone Auto Query Engine product/service | Independent lifecycle and UX for structured-data users. | Duplicates experiment, model-connection, tracking, and results capabilities already provided by AutoRAG. |
| Direct per-database SQLAlchemy / JDBC connectors in KFP pods | Broadest legacy database support without Flight SQL. | Duplicates credential, driver, and TLS management already solved by DCH; bloats pipeline container images with proprietary drivers; inconsistent cursor/type serialization. |
| Zero-shot / static Text-to-SQL agent without AutoML optimization | Faster initial implementation; no offline profiling or trial search needed. | Low accuracy on enterprise schemas (>50 tables, cryptic column names); no systematic comparison of models, schema-linking strategies, or cost/latency trade-offs. |
| Materialize all structured tables as text chunks into Vector AutoRAG | Reuses existing unstructured AutoRAG pipeline unchanged. | Fails on aggregations, joins, math, and large tables; high storage duplication and stale data synchronization issues. |

## Security and Privacy Considerations

- **Controlled SQL Execution:** All profiling, evaluation, and inference queries execute through DCH's authenticated Flight SQL interface over TLS. Query-engine components do not connect directly to databases or receive database credentials. DCH enforces connection access, while database roles enforce read-only access and applicable row/column policies.
- **Strict Read-Only Enforcement:** All generated SQL must be parsed via a deterministic SQL AST parser (e.g., `sqlglot`) prior to Flight SQL submission to reject any non-`SELECT` / non-`WITH` statements (`INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `GRANT`, `CALL`). DCH connections used by Auto Query Engine should enforce read-only database roles and Flight SQL read-only session properties.
- **Identity & RBAC Propagation:** Evaluation runs authorize the submitting user and run identity to access `dch_connection_name` and referenced model Secrets. Inference consumers authenticate to DCH in the end-user's authorization context. DCH connection authorization is separate from database identity: per-user database row/column policies require supported identity delegation or user-specific database credentials; shared connection credentials apply the shared database role's policies.
- **Data Minimization in Prompts & Logs:** Only schema metadata, bounded distinct categorical samples (excluding PII-flagged columns where annotated in DCH), and bounded query result previews are sent to the LLM. Raw database credentials never enter pipeline parameters, MLflow artifacts, or error traces.
- **Resource & Cost Guardrails:** AST rewriting injects `LIMIT` bounds on unaggregated queries, and Flight SQL session timeouts prevent runaway cartesian joins from exhausting cluster or warehouse resources.

## Risks

- **Schema Drift:** Underlying table or column changes in the DCH data source invalidate cached schema indexes and few-shot SQL examples, requiring re-profiling or re-optimization.
- **Ambiguous Business Semantics:** Technical column names (e.g., `flag_01`, `amt_net_adj`) without DCH catalog descriptions or golden SQL examples can still limit accuracy even with value sampling.
- **Flight SQL Dialect Differences:** Subtle SQL dialect variations across DCH backends (e.g., Trino vs. Postgres vs. Spark SQL) require accurate dialect detection from DCH metadata and Flight SQL `GetSqlInfo`.
- **Non-Deterministic Query Equivalence:** Equivalent queries returning differing column aliases or ordering require careful result-set normalization during Execution Accuracy evaluation.

## References

- [AutoRAG Experiment Settings](../autorag/ODH-ADR-0002-experiment-settings.md)
- [AutoRAG Pattern Inference & Consumption](../autorag/ODH-ADR-0004-rag-pattern-inference.md)
- [AutoRAG Pattern Evaluation](../autorag/ODH-ADR-0005-rag-pattern-evaluation.md)
- [AutoRAG MLflow Integration](../autorag/ODH-ADR-0006-mlflow-integration.md)
- [AutoRAG OpenAI-Compatible Model Support](../autorag/ODH-ADR-0008-openai-compatible-model-support.md)
- [Apache Arrow Flight SQL Specification](https://arrow.apache.org/docs/format/FlightSql.html)
