# RAG pattern evaluation

## Table of contents

- [Overview](#overview)
- [Judge model](#judge-model)
- [Artifacts](#artifacts)
- [Related](#related)

---

## Overview

During **`rag_templates_optimization`**, ai4rag scores each RAG pattern on a benchmark. Aggregates land in **`pattern.json`** → `evaluation`, per-row detail in **`evaluation_results.json`**, and GAM ranks patterns by **`optimization_metric`**.

AutoRAG routes metrics to internal backends; callers do not configure an **`evaluator`**.

| Metric | Backend (`evaluator`) | Question answered |
|--------|----------------------|-------------------|
| `faithfulness` | `unitxt` | Is the answer supported by retrieved context? |
| `answer_correctness` | `unitxt` | Does the answer match ground truth? |
| `context_correctness` | `unitxt` | Was retrieval sufficient? |
| `answer_relevance` | `judge` | Does the answer address the question? |
| `overall_score` | `custom` | Equal-weight mean of the four metrics |

Per-question scores are **0–1** floats; each metric entry carries **`scores`** with **`mean`, `ci_low`, `ci_high`** ([ai4rag shape](https://ibm.github.io/ai4rag/latest/user-guide/evaluation/)). All metrics run every optimization; **`optimization_metric`** (pipeline parameter, default **`overall_score`**) selects the GAM objective and is recorded in `evaluation.optimization_metric`. **`final_score`** is the mean of the metric named by `optimization_metric`.

`answer_relevance` cost scales with benchmark rows × patterns.

---

## Judge model

One judge per optimization run, used only for **`answer_relevance`**. ai4rag auto-selects it in **`search_space_preparation`** from the validated generation-model pool. The chosen **`model_id`** is recorded on the `answer_relevance` metric entry (`evaluator: judge`). Prefer a judge distinct from the **generation** model; when only one model is deployed, judge **`model_id` may equal `generation.model_id`**.

| Available models | Selection |
|------------------|-----------|
| **1** | Use that model |
| **2+** | [Calibration](#calibration) on `min(20, 10% of rows)` |

Judge rubrics and temperature use **ai4rag defaults**.

### Calibration

On a fixed calibration subset, using answers from a reference RAG configuration:

1. Score each row with **`answer_relevance`** for every candidate judge model.
2. Pick the candidate with the best **`judge_calibration_score`** (spread and stability on the subset).

**Tie-breakers:** non-generation candidate → higher search-space prep rank → lowest `model_id` lexicographically. Selection metadata is logged only.

---

## Artifacts

`pattern.json` → **`evaluation`** contains `metrics`, `optimization_metric`, and `final_score`. See [full schema](./rag_pattern_inference.md#example-patternjson).

```json
"evaluation": {
  "metrics": [
    {
      "evaluator": "unitxt",
      "name": "faithfulness",
      "description": "",
      "scores": { "mean": 0.91, "ci_low": 0.88, "ci_high": 0.94 }
    },
    {
      "evaluator": "unitxt",
      "name": "answer_correctness",
      "description": "",
      "scores": { "mean": 0.82, "ci_low": 0.78, "ci_high": 0.86 }
    },
    {
      "evaluator": "unitxt",
      "name": "context_correctness",
      "description": "",
      "scores": { "mean": 0.80, "ci_low": 0.70, "ci_high": 0.90 }
    },
    {
      "evaluator": "judge",
      "model_id": "gpt-4.1-mini",
      "name": "answer_relevance",
      "description": "",
      "scores": { "mean": 0.91, "ci_low": 0.88, "ci_high": 0.94 }
    },
    {
      "evaluator": "custom",
      "name": "overall_score",
      "description": "",
      "scores": { "mean": 0.84, "ci_low": 0.79, "ci_high": 0.89 }
    }
  ],
  "optimization_metric": "faithfulness",
  "final_score": 0.91
}
```

| Field | Description |
|-------|-------------|
| `metrics[]` | One entry per metric: `evaluator`, `name`, `description`, `scores`; `model_id` on judge entries |
| `optimization_metric` | Pipeline GAM objective used for this run |
| `final_score` | `scores.mean` of the metric named by `optimization_metric` |

---

## Related

- [RAG pattern inference](./rag_pattern_inference.md) — full `pattern.json` schema
- [AutoRAG optimization settings](./experiment_settings.md) — `optimization_metric` pipeline parameter
- [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- [ai4rag evaluation guide](https://ibm.github.io/ai4rag/latest/user-guide/evaluation/)
