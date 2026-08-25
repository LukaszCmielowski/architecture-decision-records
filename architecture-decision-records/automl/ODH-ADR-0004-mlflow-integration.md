# AutoML MLflow Integration

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-25 |
| Scope          | AutoML |
| Status         | Draft |
| Authors        | [Lukasz Cmielowski](@LukaszCmielowski) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

Integrate MLflow experiment tracking into OpenShift AI / ODH AutoML workflows implemented in [opendatahub-io/pipelines-components](https://github.com/opendatahub-io/pipelines-components) under `components/training/automl` and `pipelines/training/automl`.

## Why

AutoML pipelines currently lack a unified experiment tracking, comparison, and reproducibility layer. MLflow integration enables users to compare model runs side-by-side, trace hyperparameters and metrics across AutoGluon-trained models, and reproduce past experiments — all through a standard MLflow UI already supported by OpenShift AI.

## Goals

* Provide experiment tracking, comparison, and reproducibility for AutoML workflows via MLflow
* Use KFP-managed environment variables for opt-in MLflow integration (RHOAI 3.5+)
* Implement parent/child run hierarchy mapping AutoML pipeline executions to MLflow runs
* Extend the existing `component_stage_map` artifact with MLflow metadata for discovery and deep-linking
* Log existing training artifacts (leaderboard, metrics, feature importance, confusion matrix) to MLflow

## Non-Goals

* Native AutoGluon MLflow autologging (no `mlflow.autogluon` module exists)
* MLflow Model Registry integration (model registration is out of scope for RHOAI 3.6)

## How

### KFP MLflow integration mode

Starting with **OpenShift AI 3.5**, Data Science Pipelines provides **opt-in MLflow integration via environment variables**. When MLflow integration is enabled at the project level, the following environment variables are automatically injected into pipeline step pods:

| Environment Variable | Purpose | Example Value | Notes |
|---------------------|---------|---------------|-------|
| `MLFLOW_TRACKING_URI` | MLflow tracking server endpoint | `https://mlflow-server.example.com` | If unset, MLflow logging is disabled (optional integration) |
| `MLFLOW_WORKSPACE` | Workspace identifier for the project | `data-science-project` | Corresponds to the OpenShift AI project/namespace |
| `MLFLOW_EXPERIMENT_ID` | Auto-created experiment ID for the pipeline | `"1"` | KFP creates one experiment per pipeline; components log to this experiment directly |
| `MLFLOW_RUN_ID` | Parent run ID for the KFP pipeline execution | `"abc123..."` | KFP creates a parent run per pipeline execution; components create child runs under this |
| `MLFLOW_TRACKING_AUTH` | Authentication mechanism | `kubernetes-namespaced` | |

AutoML components check if `MLFLOW_TRACKING_URI` is set. If yes, they create **child runs** under the KFP-managed parent run (`MLFLOW_RUN_ID`) for each refitted model, using explicit `mlflow.log_params()` / `mlflow.log_metrics()` / `mlflow.log_artifact()` calls to capture AutoGluon-specific data.

### MLflow mapping model

Design for **two pipeline families** (tabular vs timeseries) with the **same conceptual mapping**.

| MLflow concept      | Proposed mapping |
|---------------------|------------------|
| **Experiment**      | KFP auto-creates one experiment per pipeline (accessible via `MLFLOW_EXPERIMENT_ID`). |
| **Parent run**      | KFP auto-creates one parent run per pipeline execution (accessible via `MLFLOW_RUN_ID`). AutoML components **resume this run** to add tags, params, and **aggregate metrics**. **Tags:** `pipeline_name`, `kfp_run_name`, `kfp_run_id`, `kfp_version`, `image`, `autogluon_version`. **Params:** `task_type` (`binary` \| `multiclass` \| `regression` \| `time_series`), `eval_metric`, `preset`, `top_n`, dataset **hashes or URIs** (non-secret). |
| **Parent Metrics**  | `best_score`, `worst_score`, `mean_score`, `num_models_trained`, `total_fit_time_seconds`. |
| **Child runs**      | **One child run per leaderboard row / refitted model** (each `name` in `model_names` or equivalent for timeseries), created as nested runs under the KFP parent. Enables side-by-side comparison in MLflow UI. Params: `model_type`, `stack_level`, `fit_time`, `predict_time`, `num_bag_folds` / `num_stack_levels` when exposed. |
| **Child Metrics**   | Task-specific metrics from AutoGluon leaderboard / `metrics.json` (e.g., `accuracy`, `f1`, `roc_auc`, `rmse`, `mae`). |
| **Child Artifacts** | Pointer to **`metrics`** (containing model's insights like confusion matrix etc.), pointer to trained model binaries **`predictor`**, and pointer to **`notebook`**. |

### Implementation approach

MLflow does **not** have native AutoGluon support. All tracking is implemented via **explicit MLflow API calls** following MLflow's nested run pattern (similar to GridSearchCV with parent-child structure). Model logging would require a custom `mlflow.pyfunc` wrapper since AutoGluon predictors cannot be logged with standard MLflow flavors.

### Alignment with AutoGluon-native logging

AutoGluon can integrate with experiment trackers in some setups. Revisit **native AutoGluon callbacks** to stream out results iteratively (as soon as a model is trained the data should be logged).

### Extending `component_stage_map.json`

AutoML pipelines already run `publish_component_stage_map` as the first task. That component writes `component_stage_map.json` with the static component/stage/step catalog plus runtime fields `kfp_run_id` and `published_at`.

Add a top-level **`mlflow`** object when the stage map is published. Other fields (`pipeline_id`, `description`, `components`, `kfp_run_id`, `published_at`) stay as today.

**When MLflow tracking is enabled** (`MLFLOW_TRACKING_URI` set):

```json
{
  "pipeline_id": "autogluon-tabular-training-pipeline",
  "description": "Tabular AutoGluon pipeline: load and split data, train and refit models, build leaderboard.",
  "components": [ "..." ],
  "published_at": "2026-05-19T12:00:00Z",
  "mlflow": {
     "tracking_uri": "https://mlflow-server.example.com",
    "experiment_id": "5",
    "run_id": "a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5",
    "workspace": "data-science-project",
    "run_url": "https://mlflow-server.example.com/#/experiments/5/runs/a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5"
  }
}
```

| `mlflow` field | Type | Source | Description |
|----------------|------|--------|-------------|
| `tracking_uri` | string | `MLFLOW_TRACKING_URI` | MLflow tracking server endpoint (omitted when disabled) |
| `experiment_id` | string | `MLFLOW_EXPERIMENT_ID` | KFP-managed experiment for this pipeline |
| `run_id` | string | `MLFLOW_RUN_ID` | KFP-managed parent run for this execution |
| `workspace` | string | `MLFLOW_WORKSPACE` | OpenShift AI project / namespace |
| `run_url` | string | Computed from `tracking_uri`, `experiment_id`, `run_id` | Deep-link to MLflow UI parent run |

`publish_component_stage_map` populates the `mlflow` block from the same KFP-injected environment variables at pipeline start. No new KFP output parameter or pipeline task is required — the existing `component_stage_map` artifact remains the single dashboard join artifact. Downstream training components still resume the parent run via `MLFLOW_RUN_ID` in the pod environment; the stage map is for **discovery and deep-linking** (AutoML Dashboard, KFP UI, CI/CD).

## Alternatives

### 1. KFP automatic MLflow logging

**Discarded:** KFP's automatic logging mode captures basic metrics but cannot handle the complex artifacts AutoGluon produces (leaderboards, ensemble hierarchies, nested model metrics). AutoML requires explicit logging control to properly represent the parent/child run structure and task-specific metrics.

### 2. Custom experiment tracking solution

**Discarded:** Building a custom tracking layer would duplicate functionality already provided by MLflow, which is supported on RHOAI and provides a standard UI for experiment comparison. MLflow also offers workspace isolation, RBAC via Kubernetes auth plugin, and is a known interface for data scientists.

## Risks

* **MLflow has no native AutoGluon support** — all tracking requires explicit API calls, increasing implementation and maintenance burden. *Mitigation: Follow patterns established by MLflow's scikit-learn and XGBoost integrations; wrap AutoGluon predictors via `mlflow.pyfunc` if model logging is needed.*
* **Environment variable dependency** — integration relies on KFP-injected environment variables being present and correct. *Mitigation: Components gracefully degrade when `MLFLOW_TRACKING_URI` is unset; the `mlflow` block in `component_stage_map.json` reflects the actual state.*

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| AutoML team                   |                  |            | Yes |
| Data Science Pipelines        |                  |            | Yes |
| MLflow integration (RHOAI)    | Matt Prahl, Humair Khan |  | Yes |
| AutoML Dashboard              |                  |            | Yes |

## References

* Component stage map publisher: [opendatahub-io/pipelines-components — `component_stage_map_publisher`](https://github.com/opendatahub-io/pipelines-components/tree/main/components/training/automl/component_stage_map_publisher)
* Upstream AutoML components: [opendatahub-io/pipelines-components — `components/training/automl`](https://github.com/opendatahub-io/pipelines-components/tree/main/components/training/automl)
* Upstream AutoML pipelines: [opendatahub-io/pipelines-components — `pipelines/training/automl`](https://github.com/opendatahub-io/pipelines-components/tree/main/pipelines/training/automl)
* End-user examples (RH): [red-hat-ai-examples — `examples/automl`](https://github.com/red-hat-data-services/red-hat-ai-examples/tree/main/examples/automl)
* Sample notebook (mocked MLflow data): [mlflow_mocks.ipynb](https://github.com/LukaszCmielowski/prototypes/blob/main/AutoML/mlflow_integration/mlflow_mocks.ipynb)
* MLflow on RHOAI Integration Guide (internal): Contact Matt Prahl or Humair Khan in `#wg-openshift-ai-mlflow-integration`
* MLflow Operator: [opendatahub-io/mlflow-operator](https://github.com/opendatahub-io/mlflow-operator)
* MLflow Workspaces documentation: [MLflow 3.10 release notes](https://github.com/mlflow/mlflow/releases/tag/v3.10.0)
* MLflow RBAC authorization plugin: [Kubernetes auth plugin documentation](https://mlflow.org/docs/latest/auth/index.html#kubernetes-authorization)
* MLflow Python Function (custom models): [MLflow pyfunc documentation](https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#creating-custom-pyfunc-models)
* MLflow Tracking: [MLflow Tracking documentation](https://mlflow.org/docs/latest/tracking.html)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
