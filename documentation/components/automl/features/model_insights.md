# Model insights (artifacts and metrics)

Artifacts produced under each refitted model directory (`{model_name}_FULL/`) by the AutoML training pipelines. Confirm filenames and schemas on your **pipelines-components** tag.

## Table of contents

- [Tabular pipeline](#tabular-pipeline)
  - [Per refitted model (`{model_name}_FULL`)](#per-refitted-model-model_name_full)
  - [Example: `model.json` (tabular)](#example-modeljson-tabular)
  - [`inference.input_data_schema` (tabular KServe scoring)](#inferenceinput_data_schema-tabular-kserve-scoring)
  - [Example: `curves.json` (binary classification)](#example-curvesjson-binary-classification)
  - [Example: `curves.json` (multiclass, one-vs-rest)](#example-curvesjson-multiclass-one-vs-rest)
- [Timeseries pipeline](#timeseries-pipeline)
  - [Per refitted model (`{model_name}_FULL`)](#per-refitted-model-model_name_full-1)
  - [Example: `model.json` (time series)](#example-modeljson-time-series)
  - [`inference.input_data_schema` (time-series KServe scoring)](#inferenceinput_data_schema-time-series-kserve-scoring)
  - [Example: `back_testing.json` (time series)](#example-back_testingjson-time-series)
  - [Visualization use cases](#visualization-use-cases)

---

## Tabular pipeline

One combined **`Model`** artifact from **`autogluon_models_training`** (`models_artifact.path`); then **`HTML`** from **`autogluon_leaderboard_evaluation`**.

### Per refitted model (`{model_name}_FULL`)

| Path | Content |
|------|---------|
| `metrics/metrics.json` | Test-split **`evaluate_predictions`**; non-finite values stripped for KFP serialization. |
| `metrics/feature_importance.json` | **`feature_importance`** (test split, `subsample_size=2000`) as dict. |
| `metrics/confusion_matrix.json` | **Classification only** (`binary` / `multiclass`); omitted for **regression**. |
| `metrics/curves.json` | **Classification only** — ROC and precision-recall curve points/thresholds for visualization (same applicability as `confusion_matrix.json`). |
| `notebooks/automl_predictor_notebook.ipynb` | Embedded regression/classification template with run, pipeline, model, and sample-row placeholders. |
| `predictor/` | **`clone_for_deployment`** export for that model. |
| `model.json` | **`name`**, **`location`**, **`metrics.test_data`**, **`inference.input_data_schema`** (tabular feature contract for KServe AutoGluon scoring). See [Example: `model.json` (tabular)](#example-modeljson-tabular). |

**`metadata`:** `model_names` (JSON **string** list, KFP workaround) and **`context`** (`task_type`, `label_column`, `model_config`, `data_config`, `models` mirroring each `model.json`). Returned **`eval_metric`** wires the leaderboard sort column.

**`curves.json` alignment:** serializes `sklearn.metrics.roc_curve()` / `precision_recall_curve()` outputs from probabilities via `predictor.predict_proba(X_test)`. **Regression** runs do not emit this file.

### Example: `model.json` (tabular)

Written by **`autogluon_models_training`** under each `{model_name}_FULL/` directory. The on-disk file matches the corresponding entry in `models_artifact.metadata.context.models`, so downstream consumers can read metadata from the filesystem without relying on KFP artifact metadata propagation.

| Field | Type | Description |
|-------|------|-------------|
| **`name`** | `str` | Refitted model id with `_FULL` suffix (e.g. `"LightGBM_BAG_L1_FULL"`). |
| **`location`** | `dict` | Paths **relative to** `models_artifact.path`: `model_directory`, `predictor`, `notebook`, `metrics`. |
| **`metrics.test_data`** | `dict` | Metric name → value from `evaluate_predictions` on the test split (same content as `metrics/metrics.json`; non-finite values stripped). |
| **`inference`** | `dict` | Deploy / scoring contract. Contains **`input_data_schema`** — see [`inference.input_data_schema` (tabular KServe scoring)](#inferenceinput_data_schema-tabular-kserve-scoring). |

```json
{
  "name": "LightGBM_BAG_L1_FULL",
  "location": {
    "model_directory": "LightGBM_BAG_L1_FULL",
    "predictor": "LightGBM_BAG_L1_FULL/predictor",
    "notebook": "LightGBM_BAG_L1_FULL/notebooks/automl_predictor_notebook.ipynb",
    "metrics": "LightGBM_BAG_L1_FULL/metrics"
  },
  "metrics": {
    "test_data": {
      "root_mean_squared_error": 0.42,
      "r2": 0.85
    }
  },
  "inference": {
    "input_data_schema": [
      { "name": "bedrooms", "datatype": "INT64", "shape": [-1] },
      { "name": "sqft", "datatype": "FP64", "shape": [-1] },
      { "name": "location", "datatype": "BYTES", "shape": [-1] }
    ]
  }
}
```

**`location` keys:**

| Key | Points to |
|-----|-----------|
| `model_directory` | `{name}/` root for this refit |
| `predictor` | `{name}/predictor` — `clone_for_deployment` export |
| `notebook` | `{name}/notebooks/automl_predictor_notebook.ipynb` |
| `metrics` | `{name}/metrics/` directory (`metrics.json`, feature importance, curves, …) |

**Relationship to artifact metadata:** each `context.models[]` entry is the same JSON object as that model’s `model.json` (including `inference` when present). `model_names` is a JSON-encoded string list of those names (KFP metadata workaround).

### `inference.input_data_schema` (tabular KServe scoring)

`inference.input_data_schema` describes the **tabular feature tensors** clients must send when scoring a model deployed with the **KServe AutoGluon ServingRuntime** (`modelFormat.name: autogluon`). It mirrors the runtime’s v2 model-metadata inputs (`GET /v2/models/{name}` / `get_input_types()` in the [KServe AutoGluon server](https://github.com/kserve/kserve/tree/master/python/autogluonserver)).

For time-series models the same path is used with a **different shape** — see [`inference.input_data_schema` (time-series KServe scoring)](#inferenceinput_data_schema-time-series-kserve-scoring).

**Purpose:** Dashboard, notebooks, and other consumers can build valid score payloads from `model.json` without loading AutoGluon or calling the live endpoint first. After deploy, the runtime still derives the same contract from the loaded `TabularPredictor`.

**Source (from the trained predictor):**

| Source | Role |
|--------|------|
| `predictor.features()` | Ordered feature / column names (one schema entry per feature) |
| `predictor.feature_metadata.type_map_raw` | AutoGluon raw type per feature → mapped to a KServe v2 `datatype` |

**Per-feature object:**

| Field | Type | Description |
|-------|------|-------------|
| **`name`** | `str` | Feature column name; must match training columns and v2 `inputs[].name` |
| **`datatype`** | `str` | Open Inference Protocol / KServe v2 tensor datatype (see mapping below) |
| **`shape`** | `list[int]` | Always `[-1]` — batch dimension; each feature tensor is shape `[batch]` (or `[batch, 1]`, flattened by the server) |

**AutoGluon raw type → KServe v2 `datatype` (same mapping as the AutoGluon server):**

| AutoGluon `type_map_raw` | `datatype` |
|--------------------------|------------|
| `int`, `int8`…`int64`, `uint*` | `INT64` |
| `float`, `float16`…`float64` | `FP64` |
| `bool` / `boolean` | `BOOL` |
| `category`, `object`, `text`, `datetime`, unknown | `BYTES` |

**Using the schema to score**

**REST v1** (`POST /v1/models/{name}:predict`) — list of row objects; keys = feature `name`s. Each value must be a **one-element list** (KServe builds `pd.DataFrame(instance)` per row; bare scalars raise).

```bash
curl -X POST "<DEPLOYMENT_URL>/v1/models/<MODEL_NAME>:predict" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TOKEN>" \
  -d '{
    "instances": [
      { "bedrooms": [3], "sqft": [1200.0], "location": ["urban"] },
      { "bedrooms": [2], "sqft": [900.0], "location": ["suburban"] }
    ]
  }'
```

Alternatively, v1 accepts a list of positional rows (feature order = `inference.input_data_schema` / `predictor.features()`), as in the KServe AutoGluon e2e fixtures.

**REST v2** — one input tensor per feature, names/datatypes/order aligned with `inference.input_data_schema` (batch size 2 in this example):

```json
{
  "inputs": [
    { "name": "bedrooms", "shape": [2], "datatype": "INT64", "data": [3, 2] },
    { "name": "sqft", "shape": [2], "datatype": "FP64", "data": [1200.0, 900.0] },
    { "name": "location", "shape": [2], "datatype": "BYTES", "data": ["urban", "suburban"] }
  ]
}
```

**Notes:**

- Tabular `inference.input_data_schema` is a **list** of `{name, datatype, shape}`. Time-series uses an **object** under the same path (see below) — do **not** wrap time-series values in one-element lists.
- Classification may return labels by default, or class probabilities when the runtime has `PREDICT_PROBA=true` (see AutoGluon server docs).
- `storageUri` for KServe must point at the `clone_for_deployment` **`predictor/`** directory referenced by `location.predictor`, not at `model.json` itself.

### Example: `curves.json` (binary classification)

```json
{
  "task_type": "binary",
  "positive_class": 1,
  "num_samples": 2000,
  "num_positive": 1024,
  "num_negative": 976,
  "roc_curve": {
    "auc": 0.9821,
    "fpr": [0.0, 0.0, 0.0123, 0.0456, 0.0891, 0.1234, 0.2345, 0.4567, 0.7890, 1.0],
    "tpr": [0.0, 0.2345, 0.5678, 0.7890, 0.8901, 0.9345, 0.9678, 0.9890, 0.9987, 1.0],
    "thresholds": ["inf", 0.9876, 0.8765, 0.7654, 0.6543, 0.5432, 0.4321, 0.3210, 0.2109, 0.1098]
  },
  "precision_recall_curve": {
    "average_precision": 0.9567,
    "precision": [0.5120, 0.6789, 0.7890, 0.8456, 0.9012, 0.9345, 0.9678, 0.9890, 1.0],
    "recall": [1.0, 0.9876, 0.9456, 0.8901, 0.8234, 0.7456, 0.6012, 0.4567, 0.0],
    "thresholds": [0.1234, 0.2345, 0.3456, 0.4567, 0.5678, 0.6789, 0.7890, 0.8901],
    "baseline_precision": 0.512
  }
}
```

**Schema notes (aligned with `sklearn.metrics`):**

**Common metadata:**
- **`task_type`**: "binary" or "multiclass"
- **`num_samples`**, **`num_positive`**, **`num_negative`**: Dataset statistics (binary only)

**ROC curve** (from `sklearn.metrics.roc_curve()`):
- **`fpr`**: False positive rate array, increasing from 0 to 1; **directly from** `roc_curve(y_true, y_pred_proba[:, 1])[0]`
- **`tpr`**: True positive rate array, increasing from 0 to 1; **directly from** `roc_curve()[1]`
- **`thresholds`**: Decreasing decision thresholds starting with `"inf"` (scikit-learn 1.3+); **directly from** `roc_curve()[2]`
- **`auc`**: Area under ROC curve via `sklearn.metrics.roc_auc_score()`
- Note: `len(fpr) == len(tpr) == len(thresholds)`

**Precision-Recall curve** (from `sklearn.metrics.precision_recall_curve()`):
- **`precision`**: Starts at baseline (class balance), ends at 1.0; **directly from** `precision_recall_curve(y_true, y_pred_proba[:, 1])[0]`
- **`recall`**: Starts at 1.0, ends at 0.0; **directly from** `precision_recall_curve()[1]`
- **`thresholds`**: Increasing array; **directly from** `precision_recall_curve()[2]`
- **`average_precision`**: Area under PR curve via `sklearn.metrics.average_precision_score()`
- **`baseline_precision`**: No-skill baseline = `num_positive / num_samples`
- Note: `len(thresholds) == len(precision) - 1` per scikit-learn convention

**AutoGluon note**: Use `predictor.predict_proba(X_test)` to obtain probability scores for both curves

**Visualization:**

```text
ROC Curve - WeightedEnsemble_L3 (Binary Classification)
AUC = 0.9821

TPR │
1.0 ┤                    ●●●●
    │                 ●●●
    │              ●●●
0.8 ┤           ●●●
    │        ●●●
    │      ●●
0.6 ┤    ●●
    │   ●
    │  ●
0.4 ┤ ●
    │●
    │
0.2 ┤●
    │
    ●────────────────────────────────
0.0 └─────────────────────────────── FPR
    0.0   0.2   0.4   0.6   0.8   1.0

● Model (AUC=0.98)
─ Random Classifier (AUC=0.50)

Best Operating Point: TPR=0.89, FPR=0.09 (threshold=0.65)
```

### Example: `curves.json` (multiclass, one-vs-rest)

```json
{
  "task_type": "multiclass",
  "strategy": "ovr",
  "num_classes": 3,
  "classes": [0, 1, 2],
  "num_samples": 3000,
  "roc_curve": {
    "auc_macro": 0.9456,
    "auc_weighted": 0.9512,
    "per_class": {
      "0": {
        "auc": 0.9821,
        "fpr": [0.0, 0.0234, 0.0567, 0.1234, 0.3456, 0.6789, 1.0],
        "tpr": [0.0, 0.4567, 0.7890, 0.9012, 0.9678, 0.9945, 1.0],
        "thresholds": ["inf", 0.8901, 0.7654, 0.6321, 0.4567, 0.2345, 0.1012],
        "support": 1034
      },
      "1": {
        "auc": 0.9234,
        "fpr": [0.0, 0.0345, 0.0789, 0.1567, 0.3890, 0.7123, 1.0],
        "tpr": [0.0, 0.5123, 0.8012, 0.9123, 0.9701, 0.9923, 1.0],
        "thresholds": ["inf", 0.9012, 0.7890, 0.6543, 0.4789, 0.2567, 0.1234],
        "support": 987
      },
      "2": {
        "auc": 0.9312,
        "fpr": [0.0, 0.0212, 0.0678, 0.1456, 0.3678, 0.6912, 1.0],
        "tpr": [0.0, 0.4890, 0.7923, 0.9089, 0.9656, 0.9934, 1.0],
        "thresholds": ["inf", 0.8765, 0.7543, 0.6234, 0.4512, 0.2389, 0.1056],
        "support": 979
      }
    }
  },
  "precision_recall_curve": {
    "average_precision_macro": 0.9234,
    "average_precision_weighted": 0.9312,
    "per_class": {
      "0": {
        "average_precision": 0.9567,
        "precision": [0.3456, 0.5678, 0.7890, 0.8901, 0.9456, 0.9789, 1.0],
        "recall": [1.0, 0.9876, 0.9234, 0.8567, 0.7456, 0.5678, 0.0],
        "thresholds": [0.1012, 0.2345, 0.4123, 0.5678, 0.6901, 0.8234],
        "baseline_precision": 0.3456
      },
      "1": {
        "average_precision": 0.9123,
        "precision": [0.3234, 0.5456, 0.7678, 0.8789, 0.9312, 0.9701, 1.0],
        "recall": [1.0, 0.9890, 0.9345, 0.8678, 0.7567, 0.5890, 0.0],
        "thresholds": [0.0987, 0.2123, 0.3987, 0.5456, 0.6789, 0.8123],
        "baseline_precision": 0.3234
      },
      "2": {
        "average_precision": 0.9012,
        "precision": [0.3123, 0.5234, 0.7456, 0.8678, 0.9234, 0.9656, 1.0],
        "recall": [1.0, 0.9912, 0.9401, 0.8789, 0.7678, 0.6012, 0.0],
        "thresholds": [0.0876, 0.2012, 0.3876, 0.5234, 0.6678, 0.7987],
        "baseline_precision": 0.3123
      }
    }
  }
}
```

**Schema notes (multiclass):**

**Common metadata:**
- **`strategy`**: `"ovr"` (one-vs-rest) or `"ovo"` (one-vs-one)
- **`num_classes`**, **`classes`**, **`num_samples`**: Dataset statistics
- **`per_class`**: Dict mapping class labels (as strings) to individual curves

**ROC curve:**
- **`auc_macro`**: Unweighted mean of per-class AUC scores
- **`auc_weighted`**: Weighted mean by support (class frequency)
- **`support`**: Number of samples per class (for weighted averaging)

**Precision-Recall curve:**
- **`average_precision_macro`**: Unweighted mean of per-class AP scores
- **`average_precision_weighted`**: Weighted mean by class support
- **`baseline_precision`**: Per-class no-skill baseline (class frequency)

**Visualization - ROC Curves (Multiclass):**

```text
ROC Curves - CatBoost_BAG_L2 (Multiclass, One-vs-Rest)
Macro AUC = 0.9456 | Weighted AUC = 0.9512

TPR │
1.0 ┤              ●●●●     ▲▲▲▲     ■■■■
    │           ●●●      ▲▲▲      ■■■
    │        ●●●      ▲▲▲      ■■■
0.8 ┤      ●●●     ▲▲▲     ■■■
    │    ●●●    ▲▲▲    ■■■
    │   ●●    ▲▲    ■■
0.6 ┤  ●●   ▲▲   ■■
    │ ●●  ▲▲  ■■
    │ ●  ▲  ■
0.4 ┤●  ▲ ■
    │● ▲■
    │●▲■
0.2 ┤●▲■
    │●▲■
    ●▲■─────────────────────────────
0.0 └─────────────────────────────── FPR
    0.0   0.2   0.4   0.6   0.8   1.0

● Class 0 (AUC=0.98, support=1034)
▲ Class 1 (AUC=0.92, support=987)
■ Class 2 (AUC=0.93, support=979)
─ Random Classifier (AUC=0.50)
```

**Visualization - Precision-Recall Curve (Binary):**

```text
Precision-Recall Curve - WeightedEnsemble_L3 (Binary Classification)
Average Precision = 0.9567

Precision │
     1.0 ┤                           ●
         │                         ●●
         │                       ●●
     0.8 ┤                     ●●
         │                   ●●
         │                 ●●
     0.6 ┤               ●●
         │             ●●
         │           ●●
     0.4 ┤         ●●
         │       ●●
         │     ●●
     0.2 ┤   ●●
         │ ●●
         ●●──────────────────────────
     0.0 └──────────────────────────── Recall
         1.0   0.8   0.6   0.4   0.2  0.0

● Model (AP=0.96)
─ No-Skill Baseline (precision=0.51)

Note: Recall decreases from left (1.0) to right (0.0)
High precision at low recall: threshold is very strict
```

**Visualization - Precision-Recall Curves (Multiclass):**

```text
Precision-Recall Curves - CatBoost_BAG_L2 (Multiclass, One-vs-Rest)
Macro AP = 0.9234 | Weighted AP = 0.9312

Precision │
     1.0 ┤                    ●●●  ▲▲▲  ■■■
         │                  ●●   ▲▲   ■■
         │                ●●   ▲▲   ■■
     0.8 ┤              ●●   ▲▲   ■■
         │            ●●   ▲▲   ■■
         │          ●●   ▲▲   ■■
     0.6 ┤        ●●   ▲▲   ■■
         │      ●●   ▲▲   ■■
         │    ●●   ▲▲   ■■
     0.4 ┤  ●●   ▲▲   ■■
         │ ●●  ▲▲  ■■
         │●●  ▲▲  ■■
     0.2 ┤●  ▲▲  ■■
         │● ▲▲ ■■
         ●●▲▲■■────────────────────
     0.0 └──────────────────────────── Recall
         1.0   0.8   0.6   0.4   0.2  0.0

● Class 0 (AP=0.96, baseline=0.35, support=1034)
▲ Class 1 (AP=0.91, baseline=0.32, support=987)
■ Class 2 (AP=0.90, baseline=0.31, support=979)

Per-class baselines shown as horizontal lines at respective precision values
```

## Timeseries pipeline

**`autogluon_timeseries_models_full_refit`** emits **one `Model` artifact per refit** (often **`ParallelFor`**, parallelism **1**). Selection **`leaderboard`** is not written to disk; persisted insight material is **refit** outputs plus leaderboard **HTML**.

### Per refitted model (`{model_name}_FULL`)

| Path | Content |
|------|---------|
| `metrics/metrics.json` | **`evaluate`** on held-out test data with **all** `AVAILABLE_METRICS` keys; non-finite stripped. |
| `metrics/back_testing.json` | Multi-window backtest: per-window metrics and forecast detail for best/worst series (from AutoGluon backtest / cutoff evaluation). |
| `predictor/` | Saved **`TimeSeriesPredictor`**. |
| `predictor/predictor_metadata.json` | Model id, **`prediction_length`**, **`eval_metric`**, **`target`**, **`id_column`**, **`timestamp_column`**. |
| `notebooks/automl_predictor_notebook.ipynb` | **`timeseries_notebook.ipynb`** template with run / pipeline / model / sample / column placeholders. |
| `model.json` | **`name`**, **`location`**, **`metrics.test_data`**, **`inference.input_data_schema`** (long-format column contract for KServe AutoGluon TS scoring). Optional **`location.back_testing`**. See [Example: `model.json` (time series)](#example-modeljson-time-series). |

**No** `feature_importance.json`, `confusion_matrix.json`, or `curves.json` for time-series.

### Example: `model.json` (time series)

Written by **`autogluon_timeseries_models_training`** under each `{model_name}_FULL/` directory. Same core fields as tabular (`name`, `location`, `metrics.test_data`, `inference`), plus optional `location.back_testing`. Time-series **`inference.input_data_schema`** is an object (not a feature-tensor list).

| Field | Type | Description |
|-------|------|-------------|
| **`name`** | `str` | Refitted model id with `_FULL` suffix (e.g. `"DeepAR_FULL"`). |
| **`location`** | `dict` | Relative paths: `model_directory`, `predictor`, `notebook`, `metrics`; plus **`back_testing`** when `metrics/back_testing.json` exists. |
| **`metrics.test_data`** | `dict` | Metric name → value from time-series **`evaluate`** on held-out test data (same scores as `metrics/metrics.json`; non-finite stripped). |
| **`inference`** | `dict` | Deploy / scoring contract. Contains **`input_data_schema`** — see [`inference.input_data_schema` (time-series KServe scoring)](#inferenceinput_data_schema-time-series-kserve-scoring). |

```json
{
  "name": "DeepAR_FULL",
  "location": {
    "model_directory": "DeepAR_FULL",
    "predictor": "DeepAR_FULL/predictor",
    "notebook": "DeepAR_FULL/notebooks/automl_predictor_notebook.ipynb",
    "metrics": "DeepAR_FULL/metrics",
    "back_testing": "DeepAR_FULL/metrics/back_testing.json"
  },
  "metrics": {
    "test_data": {
      "WQL": 0.2198,
      "MAPE": 11.87,
      "MASE": 0.8432,
      "RMSE": 43.21,
      "MAE": 32.56
    }
  },
  "inference": {
    "input_data_schema": {
      "protocol": "v1_json",
      "id_column": "product_id",
      "timestamp_column": "date",
      "target": "sales",
      "prediction_length": 7,
      "known_covariates_names": ["promo"],
      "instances_columns": [
        { "name": "product_id", "role": "id", "datatype": "BYTES" },
        { "name": "date", "role": "timestamp", "datatype": "BYTES" },
        { "name": "sales", "role": "target", "datatype": "FP64" }
      ],
      "known_covariates_columns": [
        { "name": "product_id", "role": "id", "datatype": "BYTES" },
        { "name": "date", "role": "timestamp", "datatype": "BYTES" },
        { "name": "promo", "role": "known_covariate", "datatype": "INT64" }
      ]
    }
  }
}
```

If backtesting was skipped or failed for that model, **`location.back_testing` is omitted**; `metrics/back_testing.json` may also be absent. When the model has no known covariates, `known_covariates_names` is `[]` and `known_covariates_columns` lists only id + timestamp (or may be omitted).

### `inference.input_data_schema` (time-series KServe scoring)

Time-series uses **REST v1 JSON only** (`POST /v1/models/{name}:predict`). v2 tensor payloads are not supported. `inference.input_data_schema` on time-series `model.json` is an **object** (not a feature-tensor list) so clients can build long-format `instances` / `known_covariates` without loading AutoGluon.

**Source:**

| Source | Role |
|--------|------|
| `TimeSeriesPredictor.target` | Target column name in history rows |
| `TimeSeriesPredictor.prediction_length` | Forecast horizon (steps of `known_covariates` per series when required) |
| `TimeSeriesPredictor.known_covariates_names` | Covariate columns required on the horizon |
| `predictor_metadata.json` / training params | `id_column`, `timestamp_column` (overridable at serve time via `AUTOGLUON_TS_ID_COLUMN` / `AUTOGLUON_TS_TIMESTAMP_COLUMN`) |

**Schema fields:**

| Field | Type | Description |
|-------|------|-------------|
| **`protocol`** | `str` | Always `"v1_json"` for TS AutoGluon ServingRuntime |
| **`id_column`** | `str` | Series id column name in request JSON |
| **`timestamp_column`** | `str` | Timestamp column name in request JSON |
| **`target`** | `str` | Target / observed value column in **`instances`** history |
| **`prediction_length`** | `int` | Horizon length; when known covariates are used, supply that many future rows per series |
| **`known_covariates_names`** | `list[str]` | Known covariate feature names (empty if none) |
| **`instances_columns`** | `list[dict]` | Columns required in each `instances[]` row: id, timestamp, target (+ any history covariates if trained with them) |
| **`known_covariates_columns`** | `list[dict]` | Columns required in each `known_covariates[]` row when `known_covariates_names` is non-empty: id, timestamp, + each known covariate |

**Column object:** `{ "name", "role", "datatype" }` where `role` is `id` | `timestamp` | `target` | `known_covariate` | `covariate`, and `datatype` uses the same KServe v2-style labels as tabular (`INT64`, `FP64`, `BOOL`, `BYTES`) for client hints — the wire format is still JSON scalars.

**Using the schema to score**

Unlike tabular v1, the server builds history with `pd.DataFrame(instances)` over the **whole list**. Values are **bare scalars** — do **not** wrap them in one-element lists.

| Payload field | Built from |
|---------------|------------|
| **`instances`** | One object per history timestamp; keys = `instances_columns[].name` |
| **`known_covariates`** | Omit if `known_covariates_names` is empty; else one object per horizon step × series; keys = `known_covariates_columns[].name` |

```bash
curl -X POST "<DEPLOYMENT_URL>/v1/models/<MODEL_NAME>:predict" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TOKEN>" \
  -d '{
    "instances": [
      { "product_id": "A", "date": "2024-01-01T00:00:00", "sales": 12.3 },
      { "product_id": "A", "date": "2024-01-02T00:00:00", "sales": 11.1 }
    ],
    "known_covariates": [
      { "product_id": "A", "date": "2024-01-03T00:00:00", "promo": 1 }
    ]
  }'
```

Response is `{"predictions": [ ... ]}` with the same id/timestamp column names plus `mean` and quantile columns from the predictor.

`storageUri` must point at the saved **`predictor/`** directory (`location.predictor`). After deploy, the runtime also reads `predictor_metadata.json` / the loaded predictor; `inference.input_data_schema` is the offline contract for UI and notebooks.

### Example: `back_testing.json` (time series)

```json
{
  "model_name": "DeepAR_FULL",
  "prediction_length": 7,
  "num_val_windows": 3,
  "eval_metric": "WQL",
  "target": "sales",
  "id_column": "product_id",
  "timestamp_column": "date",
  "per_window_metrics": [
    {
      "window_id": 0,
      "test_start": "2025-12-08",
      "test_end": "2025-12-14",
      "metrics": {
        "WQL": 0.2198,
        "MAPE": 11.87,
        "MASE": 0.8432,
        "RMSE": 43.21,
        "MAE": 32.56
      }
    },
    {
      "window_id": 1,
      "test_start": "2025-12-15",
      "test_end": "2025-12-21",
      "metrics": {
        "WQL": 0.2367,
        "MAPE": 12.45,
        "MASE": 0.8876,
        "RMSE": 46.78,
        "MAE": 35.12
      }
    },
    {
      "window_id": 2,
      "test_start": "2025-12-22",
      "test_end": "2025-12-28",
      "metrics": {
        "WQL": 0.2458,
        "MAPE": 12.98,
        "MASE": 0.8987,
        "RMSE": 47.02,
        "MAE": 35.95
      }
    }
  ],
  "series_analysis": {
    "num_series_evaluated": 1000,
    "best_performer": {
      "item_id": "series_042",
      "avg_metrics": {
        "MAPE": 2.45,
        "RMSE": 13.1,
        "MAE": 8.7,
        "WQL": 0.1234
      },
      "windows": [
        {
          "window_id": 0,
          "metrics": {
            "MAPE": 2.58,
            "RMSE": 13.8,
            "MAE": 9.1,
            "WQL": 0.1298
          },
          "forecast_data": [
            {
              "timestamp": "2025-12-08T00:00:00Z",
              "actual": 515.2,
              "predicted": 512.8,
              "lower_bound": 496.5,
              "upper_bound": 529.1
            },
            {
              "timestamp": "2025-12-08T01:00:00Z",
              "actual": 510.5,
              "predicted": 509.2,
              "lower_bound": 493.0,
              "upper_bound": 525.4
            }
          ]
        },
        {
          "window_id": 1,
          "metrics": {
            "MAPE": 2.41,
            "RMSE": 12.9,
            "MAE": 8.5,
            "WQL": 0.1245
          },
          "forecast_data": [
            {
              "timestamp": "2025-12-15T00:00:00Z",
              "actual": 520.8,
              "predicted": 519.3,
              "lower_bound": 502.9,
              "upper_bound": 535.7
            }
          ]
        },
        {
          "window_id": 2,
          "metrics": {
            "MAPE": 2.34,
            "RMSE": 12.5,
            "MAE": 8.3,
            "WQL": 0.1158
          },
          "forecast_data": [
            {
              "timestamp": "2025-12-22T00:00:00Z",
              "actual": 522.5,
              "predicted": 521.8,
              "lower_bound": 505.2,
              "upper_bound": 538.4
            }
          ]
        }
      ]
    },
    "worst_performer": {
      "item_id": "series_104",
      "avg_metrics": {
        "MAPE": 42.58,
        "RMSE": 253.4,
        "MAE": 158.7,
        "WQL": 0.8765
      },
      "windows": [
        {
          "window_id": 0,
          "metrics": {
            "MAPE": 41.23,
            "RMSE": 245.8,
            "MAE": 152.4,
            "WQL": 0.8512
          },
          "forecast_data": [
            {
              "timestamp": "2025-12-08T00:00:00Z",
              "actual": 89.4,
              "predicted": 198.7,
              "lower_bound": 168.3,
              "upper_bound": 229.1
            },
            {
              "timestamp": "2025-12-08T01:00:00Z",
              "actual": 92.1,
              "predicted": 203.5,
              "lower_bound": 172.8,
              "upper_bound": 234.2
            }
          ]
        },
        {
          "window_id": 1,
          "metrics": {
            "MAPE": 44.85,
            "RMSE": 265.3,
            "MAE": 168.2,
            "WQL": 0.9123
          },
          "forecast_data": [
            {
              "timestamp": "2025-12-15T00:00:00Z",
              "actual": 82.7,
              "predicted": 205.3,
              "lower_bound": 174.2,
              "upper_bound": 236.4
            }
          ]
        },
        {
          "window_id": 2,
          "metrics": {
            "MAPE": 41.65,
            "RMSE": 249.1,
            "MAE": 155.5,
            "WQL": 0.8659
          },
          "forecast_data": [
            {
              "timestamp": "2025-12-22T00:00:00Z",
              "actual": 85.3,
              "predicted": 201.4,
              "lower_bound": 170.8,
              "upper_bound": 232.0
            }
          ]
        }
      ]
    }
  }
}
```

**Schema notes (aligned with AutoGluon TimeSeriesPredictor API):**
- **`per_window_metrics`**: Array of metrics for each backtesting window; **obtained by iterating** `evaluate()` with different `cutoff` values or using `ExpandingWindowSplitter`
  - Example: `for cutoff in range(-num_val_windows * prediction_length, 0, prediction_length): score = predictor.evaluate(test_data, cutoff=cutoff)`
  - Returns dict with keys from `AVAILABLE_METRICS`: `{'WQL': 0.234, 'MAPE': 12.34, 'MASE': 0.876, 'RMSE': 45.67, 'MAE': 34.21}`
- **`series_analysis`**: Analysis of best and worst performing series; **derived from** `predictor.backtest_predictions()` and custom per-series metric computation (not built-in AutoGluon). Contains `best_performer` and `worst_performer` objects with full `windows` arrays including `forecast_data` for timestamp-level predictions.
- **AutoGluon methods used**:
  - `predictor.backtest_predictions(test_data, num_val_windows=3)` → list of TimeSeriesDataFrame (predictions per window)
  - `predictor.backtest_targets(test_data, num_val_windows=3)` → list of TimeSeriesDataFrame (targets per window)
  - `predictor.evaluate(test_data, cutoff=...)` → dict of metrics for specific window

**Visualization 1: Per-Window Metric Trends**

```text
Back-testing Performance - DeepAR_FULL (3 validation windows)

WQL  │
0.30 ┤
     │                                    ●
0.25 ┤                        ●          
     │                                    
0.20 ┤            ●                       
     │                                    
0.15 ┤                                    
     │                                    
0.10 ┤                                    
     │                                    
0.05 ┤                                    
     │                                    
0.00 └─────────────────────────────────────
     Window 0      Window 1      Window 2
     12/07-12/14   12/14-12/21   12/21-12/28

⚠️  Performance degradation detected: WQL increased from 0.22 to 0.25 (+11.8%)
    MAPE increased from 11.87% to 12.98% (+9.3%)
```

**Visualization 2: Forecast Timeline (Best Performer - series_042)**

```text
═══════════════════════════════════════════════════════════════════════════════
series_042 - Best Performer (Avg MAPE = 2.45%)
Actual vs Predicted across 3 backtest windows
═══════════════════════════════════════════════════════════════════════════════

Value │
  540 ┤
      │                                      ●            ●
  520 ┤            ● ●         ┊        ----●----    ┊  ---●---
      │         ───────        ┊     ───      ───   ┊ ───   ───
  500 ┤      ───       ───     ┊  ───            ───┊───
      │   ───             ───  ┊ ───                ───
  480 ┤───                   ──┴──                   ┴
      └─────────────────────────────────────────────────────────────
      Dec 7         Dec 8     │   Dec 14    Dec 15  │  Dec 21   Dec 22
                          Window 0 │             Window 1 │       Window 2
                           Test: 12/08        Test: 12/15       Test: 12/22

● Actual values    ─── Predicted values    │ Window boundaries

Window 0 (12/08): actual=[515.2, 510.5], predicted=[512.8, 509.2], MAPE=2.58%
Window 1 (12/15): actual=[520.8], predicted=[519.3], MAPE=2.41%
Window 2 (12/22): actual=[522.5], predicted=[521.8], MAPE=2.34%
```

**Visualization 3: Forecast Timeline (Worst Performer - series_104)**

```text
═══════════════════════════════════════════════════════════════════════════════
series_104 - Worst Performer (Avg MAPE = 42.58%)
Actual vs Predicted across 3 backtest windows (systematic overprediction)
═══════════════════════════════════════════════════════════════════════════════

Value │
  220 ┤              ──────────────────────────────────────────────────
      │           ───       ┊         ───       ┊         ───
  200 ┤        ───          ┊      ───          ┊      ───
      │     ───             ┊   ───             ┊   ───
  180 ┤  ───                ┊ ───               ┊ ───
      │──                   ┴──                  ┴
  160 ┤
      │
  140 ┤
      │
  120 ┤
      │
  100 ┤            ● ●         ┊        ●           ┊        ●
   80 ┤
   60 ┤
      └─────────────────────────────────────────────────────────────
      Dec 7         Dec 8     │   Dec 14    Dec 15  │  Dec 21   Dec 22
                          Window 0 │             Window 1 │       Window 2
                           Test: 12/08        Test: 12/15       Test: 12/22

● Actual values    ─── Predicted values    │ Window boundaries

Window 0 (12/08): actual=[89.4, 92.1], predicted=[198.7, 203.5], MAPE=41.23%
Window 1 (12/15): actual=[82.7], predicted=[205.3], MAPE=44.85%
Window 2 (12/22): actual=[85.3], predicted=[201.4], MAPE=41.65%

⚠️  Model consistently overpredicts by ~120 units (~140% error) across all windows
```

**Visualization 4: Per-Series Performance Summary**

```text
┌──────────────┬──────────┬──────────┬────────────────────┐
│ Item ID      │ Avg WQL  │ Avg MAPE │ Performance        │
├──────────────┼──────────┼──────────┼────────────────────┤
│ series_042   │  0.1234  │   2.45%  │ ⭐ Best Performer  │
│ series_104   │  0.8765  │  42.58%  │ ⚠️  Worst Performer│
└──────────────┴──────────┴──────────┴────────────────────┘

Best: Tight tracking, improving over time (MAPE: 2.58% → 2.34%)
Worst: Systematic overprediction, degraded in Window 1 (MAPE: 44.85%)
```

### Visualization use cases

See mocked visualizations above for each artifact type. These examples demonstrate:

**Classification Curves (`curves.json`):**
- **ROC Curve**:
  - Binary: Single curve comparing model (AUC=0.98) against random baseline (AUC=0.50)
  - Multiclass: Overlaid one-vs-rest curves showing per-class performance with support counts
  - Key insight: Identify best operating point (threshold) balancing TPR/FPR for deployment
  - Implementation: matplotlib `plt.plot(fpr, tpr)`, plotly `go.Scatter(x=fpr, y=tpr)`
- **Precision-Recall Curve**:
  - Binary: Shows model maintains high precision (>0.9) even at moderate recall (>0.6)
  - Multiclass: Per-class curves with baselines; Class 0 performs best (AP=0.96)
  - Key insight: Critical for imbalanced datasets; precision drop-off indicates threshold sensitivity
  - Implementation: matplotlib `plt.plot(recall, precision)`, add `plt.hlines()` for baseline
- **Note**: Both curves merged into single file to avoid duplicating metadata (task_type, num_samples, class labels, support counts)

**Back-testing (`back_testing.json`):**
- **Visualization 1 - Per-Window Trend**: Detects performance degradation across backtest windows (WQL increased 11.8% from window 0 to window 2)
- **Visualization 2-3 - Forecast Timelines**: Shows actual vs predicted values across all windows for best/worst performers; reveals systematic biases (e.g., series_104 consistently overpredicts by ~140%)
- **Visualization 4 - Performance Summary**: Quick comparison table showing best/worst series with average metrics
- **Key insights**: 
  - Best performer (series_042): Tight tracking, improving trend (MAPE: 2.58% → 2.34%)
  - Worst performer (series_104): Systematic overprediction, degraded in Window 1 (MAPE: 44.85%)
  - Timeline view enables identification of persistent bias vs random errors
- **Implementation**: 
  - Per-window trends: pandas `df.plot()` with window_id on x-axis
  - Forecast timelines: matplotlib/plotly line charts with dual series (actual=scatter, predicted=line), vertical lines for window boundaries
  - Similar to AutoGluon's `predictor.plot(test_data, predictions, max_history_length=...)` but using `forecast_data` arrays from JSON

**Integration with Jupyter Notebooks:**

The `automl_predictor_notebook.ipynb` template (per model) can automatically load these JSON artifacts and render interactive visualizations.

---

**Implementation References:**
- [AutoGluon Forecasting In-Depth Tutorial](https://auto.gluon.ai/dev/tutorials/timeseries/forecasting-indepth.html) (multi-window backtesting examples)
- [AutoGluon Forecasting Notebook (Colab)](https://colab.research.google.com/github/autogluon/autogluon/blob/master/docs/tutorials/timeseries/forecasting-indepth.ipynb)
- [TabularPredictor.evaluate_predictions API](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.evaluate_predictions.html)
- [TimeSeriesPredictor.plot API](https://auto.gluon.ai/dev/api/autogluon.timeseries.TimeSeriesPredictor.plot.html) (built-in visualization method)

---
