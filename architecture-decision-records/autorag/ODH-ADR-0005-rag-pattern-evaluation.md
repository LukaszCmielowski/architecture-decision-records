# Open Data Hub - AutoRAG Pattern Evaluation

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-31 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1425](https://redhat.atlassian.net/browse/RHAISTRAT-1425) · [RHAISTRAT-2623](https://redhat.atlassian.net/browse/RHAISTRAT-2623) · [RHOAIENG-88692](https://redhat.atlassian.net/browse/RHOAIENG-88692) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) |

## What

This ADR documents how AutoRAG / ai4rag scores RAG patterns during rag_templates_optimization: Unitxt and Ragas metric backends, the GAM `optimization_metric` `{name, evaluator}` object, aggregates in pattern.json, row-level evaluation_results.json, and document identity by object key (`document_key`). Benchmark input fields live in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#benchmark-json).

## Why

Comparable, standardized metrics are required for GAM ranking and for users to trust pattern selection. Unitxt and Ragas both emit `faithfulness`, so the objective and every artifact row must carry **name and evaluator**. The evaluation contract must stay aligned with pipeline parameters and Dashboard leaderboards.

## Goals

* Define the metric set computed every optimization run (`unitxt`, `ragas`, `custom`) and that callers select only the GAM objective from that set
* Specify GAM `optimization_metric` as `{name, evaluator}`
* Specify evaluation aggregates in pattern.json and per-row evaluation_results.json
* Identify retrieved context by `document_key` (object key)

## Non-Goals

* Search-space dimensions and pipeline parameters other than `optimization_metric` (see ODH-ADR-0002)
* Optional DSPy prompt pre-optimization (see ODH-ADR-0006)

## How

## Table of contents

- [Overview](#overview)
- [Metric catalog](#metric-catalog)
- [optimization_metric](#optimization_metric)
- [Ragas runtime](#ragas-runtime)
- [evaluation_results.json](#evaluation_resultsjson)
- [Related](#related)

## Overview

During **`rag_templates_optimization`**, ai4rag scores each RAG pattern on a benchmark.

* **All catalog metrics from all evaluators are computed every run** (Unitxt, Ragas, and `overall_score`). Callers do not enable or disable evaluators or individual metrics.
* The only user choice is **`optimization_metric`**: which one catalog pair GAM uses to rank patterns. That pair must be from the full set. Default: `{ "name": "overall_score", "evaluator": "custom" }`.
* Aggregates land in **`pattern.json`** → `evaluation.metrics[]` ([schema](./ODH-ADR-0004-rag-pattern-inference.md#patternjson)).
* Per-row detail lands in **`evaluation_results.json`**.

Per-question values are **0–1** floats. At pattern level, each `metrics[]` entry carries **`scores`** with **`mean`, `ci_low`, `ci_high`**. The matching `{name, evaluator}` pair is marked `optimization_metric: true`; GAM uses that entry's `scores.mean`.

---

## Metric catalog

This full set is computed every optimization run. The user selects only which pair is the GAM objective ([optimization_metric](#optimization_metric)).

**Unitxt** (`evaluator: "unitxt"`)

| name | Question answered |
|------|-------------------|
| `faithfulness` | Is the answer supported by retrieved context? |
| `answer_correctness` | Does the answer match ground truth? |
| `context_correctness` | Was retrieval sufficient vs ground-truth docs? |

**Ragas** (`evaluator: "ragas"`)

| name | Question answered |
|------|-------------------|
| `faithfulness` | Grounded in retrieved context (Ragas algorithm, not Unitxt) |
| `answer_relevancy` | On-topic vs the question (needs embeddings) |
| `context_precision` | Are relevant contexts ranked high? |
| `context_recall` | How much of the ground-truth answer is in retrieved context? |

**Derived** (`evaluator: "custom"`)

| name | Question answered |
|------|-------------------|
| `overall_score` | Equal-weight mean of every other metric that ran |

Unitxt `faithfulness` and Ragas `faithfulness` share a **name**. Disambiguate with **`evaluator`**. Do not rename artifact fields.

---

## optimization_metric

Pipeline and experiment input ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)):

```json
{
  "optimization_metric": {
    "name": "faithfulness",
    "evaluator": "ragas"
  }
}
```

Rules:

* Every run still computes the **full catalog** (all Unitxt, all Ragas, and `overall_score`). `optimization_metric` does not subset evaluation.
* The chosen `{name, evaluator}` must be a pair from the [catalog](#metric-catalog) and must match exactly one `evaluation.metrics[]` entry.
* That entry is the only one with `optimization_metric: true`. GAM uses its `scores.mean`.
* A name without `evaluator` is invalid.

Default:

```json
{ "name": "overall_score", "evaluator": "custom" }
```

---

## Ragas runtime

Ragas calls use the same MaaS client as generation (`MAAS_BASE_URL`, `MAAS_API_KEY`).

* Evaluating LLM: first search-space generation model.
* Embeddings (`answer_relevancy`): first search-space embedding model.
* `model_id` on Ragas `metrics[]` entries records those models when ai4rag emits them.
* Cost scales with benchmark rows × patterns × four Ragas metrics (plus embedding calls for `answer_relevancy`).
* Failed or slow samples yield a `null` score; they do not fail the whole pattern.
* The AUTORAG image includes the ai4rag Ragas dependency.

---

## evaluation_results.json

Each pattern subdirectory under **`rag_patterns/<pattern_name>/`** contains **`evaluation_results.json`**: a JSON **array** with one object per benchmark row. Use it to inspect failures, compare retrieval quality across patterns, or audit scores. Run-level aggregates (`scores.mean`, `ci_low`, `ci_high`) live in `pattern.json` → `evaluation.metrics[]` ([example](./ODH-ADR-0004-rag-pattern-inference.md#example-patternjson)), computed from `metrics[].score` across these rows. Sibling files: [ODH-ADR-0004 — Pattern artifacts](./ODH-ADR-0004-rag-pattern-inference.md#pattern-artifacts).

| Field | Description |
|-------|-------------|
| `question` | Benchmark question text |
| `correct_answers` | Ground-truth answers from the benchmark JSON |
| `answer` | Generated answer for this pattern |
| `answer_contexts[]` | Retrieved chunks: `text`, `document_key` (full object key / path, not a filename) |
| `metrics[]` | Per-metric scores for this row: `name`, `evaluator`, `score` (**0–1** float). `name` + `evaluator` match `evaluation.metrics[]` in `pattern.json` |

Ground-truth document keys stay on the benchmark input ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#benchmark-json)). Pattern / leaderboard source labels use the same object key as `document_key`. Two locations with the same basename remain distinct because `document_key` is the full object path.

KFP **`evaluation_results.json`** is the source of truth for row-level scores and retrieved context.

```json
[
  {
    "question": "What warranty period applies to the XR-200 controller?",
    "correct_answers": ["The XR-200 controller has a 24-month warranty."],
    "answer": "The XR-200 controller is covered by a 24-month warranty from the date of purchase.",
    "answer_contexts": [
      {
        "text": "XR-200 Controller — Warranty: 24 months from purchase date.",
        "document_key": "product-manuals/xr-200-manual.pdf"
      },
      {
        "text": "Register your XR-200 within 30 days to activate warranty coverage.",
        "document_key": "policies/warranty-policy.pdf"
      }
    ],
    "metrics": [
      { "name": "faithfulness", "evaluator": "unitxt", "score": 0.94 },
      { "name": "answer_correctness", "evaluator": "unitxt", "score": 0.88 },
      { "name": "context_correctness", "evaluator": "unitxt", "score": 0.82 },
      { "name": "faithfulness", "evaluator": "ragas", "score": 0.85 },
      { "name": "answer_relevancy", "evaluator": "ragas", "score": 0.91 },
      { "name": "context_precision", "evaluator": "ragas", "score": 0.80 },
      { "name": "context_recall", "evaluator": "ragas", "score": 0.93 },
      { "name": "overall_score", "evaluator": "custom", "score": 0.88 }
    ]
  },
  {
    "question": "Which firmware versions support remote diagnostics?",
    "correct_answers": ["Firmware 3.2 and later supports remote diagnostics."],
    "answer": "Remote diagnostics require firmware 3.2 or newer.",
    "answer_contexts": [
      {
        "text": "Remote diagnostics are available starting in firmware release 3.2.",
        "document_key": "release-notes/release-notes-3.2.pdf"
      }
    ],
    "metrics": [
      { "name": "faithfulness", "evaluator": "unitxt", "score": 0.97 },
      { "name": "answer_correctness", "evaluator": "unitxt", "score": 0.85 },
      { "name": "context_correctness", "evaluator": "unitxt", "score": 0.78 },
      { "name": "faithfulness", "evaluator": "ragas", "score": 0.87 },
      { "name": "answer_relevancy", "evaluator": "ragas", "score": 0.90 },
      { "name": "context_precision", "evaluator": "ragas", "score": 0.76 },
      { "name": "context_recall", "evaluator": "ragas", "score": 0.92 },
      { "name": "overall_score", "evaluator": "custom", "score": 0.86 }
    ]
  }
]
```

---

## Related

- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md) — full `pattern.json` schema and artifact layout
- [RAG templates](./ODH-ADR-0003-rag-templates.md) — current simple template vs planned Graph RAG
- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md) — `optimization_metric` `{name, evaluator}`; corpus list; benchmark JSON (`correct_answer_document_keys`)
- [Prompt tuning](./ODH-ADR-0006-prompt-tuning.md) — optional DSPy prompt pre-optimization before GAM
- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
- [ai4rag evaluation guide](https://github.com/IBM/ai4rag/blob/main/docs/user-guide/evaluation.md)
- [pipelines-components PR #204](https://github.com/opendatahub-io/pipelines-components/pull/204) — Ragas wiring on `rag_templates_optimization`
- [ai4rag PR #126](https://github.com/IBM/ai4rag/pull/126) — `RagasEvaluator`
