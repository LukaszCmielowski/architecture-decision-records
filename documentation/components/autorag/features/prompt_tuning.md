# Prompt tuning (DSPy pre-optimization)

DSPy-based **prompt pre-optimization** generates data-driven system-prompt candidates before AutoRAG’s main GAM search. Optimized prompts expand the generation search space so retrieval/chunking HPO is not limited to hand-crafted defaults.

## Table of contents

- [Overview](#overview)
- [Two-stage flow](#two-stage-flow)
- [Pre-check behavior](#pre-check-behavior)
- [Configuration](#configuration)
- [Metric](#metric)
- [Integration with GAM](#integration-with-gam)
- [Related](#related)

---

## Overview

Today’s GAM loop explores chunking, embedding, retrieval, and generation settings while **system prompts stay fixed**. Prompt quality then caps RAG performance even when retrieval is well tuned.

**Prompt tuning** adds an optional **pre-check** stage in ai4rag:

1. Subsample the benchmark
2. Run **DSPy MIPROv2** to propose optimized instruction / system-message candidates
3. Inject those candidates (plus defaults) into the GAM prompt search space

Callers can skip the pre-check and keep default prompts only (backward compatible).

---

## Two-stage flow

```text
Input: corpus sample + benchmark
        │
        ▼
┌───────────────────────────────────────┐
│ Stage 1 — Prompt pre-check (optional) │
│  • Subsample ~10–20 benchmark rows    │
│  • Baseline: ai4rag default prompts   │
│  • Optimize: DSPy MIPROv2             │
│  • Output: N prompt candidates        │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Stage 2 — GAM (existing)              │
│  • Search: chunking × retrieval × …   │
│            × (default + tuned prompts)│
│  • Score: full benchmark metrics      │
│  • Output: ranked RAG patterns        │
└───────────────────────────────────────┘
```

Stage 1 is intentionally small and short (~10–15 minutes in prototypes) so Stage 2 still owns full-corpus-sample evaluation and pattern emission (`pattern.json`).

---

## Pre-check behavior

| Step | Behavior |
|------|----------|
| **Subsample** | Select a stratified subset of benchmark questions (typical size 10–20) |
| **Baseline** | Start from ai4rag default system / instruction templates |
| **Module** | DSPy RAG module with an explicit **instruction** field (e.g. `InstructableRAG` / `ChainOfThought` over `context, question → answer`) |
| **Optimizer** | **MIPROv2** (`auto` intensity such as `light` or `medium`) |
| **Output** | `num_candidates` optimized prompt variants (typical 2–5), each with a pre-check quality score |

Prototype work showed concise, extraction-focused instructions often scoring higher under LLM-as-a-judge than verbose ones—even when token-overlap metrics preferred longer answers. Pre-check therefore uses a **quality judge**, not token overlap, as the optimization metric.

---

## Configuration

| Setting | Typical range | Role |
|---------|---------------|------|
| `subsample_size` | 10–20 | Benchmark rows used for MIPROv2 |
| `num_candidates` | 2–5 | Optimized prompts added to the GAM search space |
| `auto_mode` | `light` / `medium` | MIPROv2 optimization intensity |
| Enable / disable | feature flag or API option | Skip Stage 1 for default-prompt-only runs |

Generation and judge calls use the same **OGX** client / foundation-model pool as the main experiment. Dependencies include **`dspy-ai`** and **`optuna`** (required by MIPROv2).

---

## Metric

Pre-check ranks prompt candidates with an **LLM-as-a-Judge** score (normalized **0–1**), aligned with AutoRAG’s judge-style evaluation rather than surface token overlap.

| Setting | Guidance |
|---------|----------|
| Temperature | `0.0` (deterministic scoring) |
| Max tokens | Generous enough for score extraction (prototype used ~500; avoid tiny caps that truncate thinking/score text) |
| Scale | Internal 1–5 (or equivalent) mapped to **0.0–1.0** |

Full-benchmark metrics for Stage 2 remain those in [RAG pattern evaluation](./rag_pattern_evaluation.md) (`faithfulness`, `answer_correctness`, `context_correctness`, `answer_relevance`, `overall_score`).

---

## Integration with GAM

- Pre-check output is a list of prompt templates merged with **defaults** before GAM explores the product search space.
- GAM still selects the best overall pattern (retrieval + chunking + generation, including which prompt won).
- Winning generation text lands in **`pattern.json`** → `settings.generation` (and the frozen `inference.responses_template`) like any other trial—see [RAG pattern inference](./rag_pattern_inference.md).
- Pre-check can run standalone for inspection, or as part of an end-to-end experiment helper that chains Stage 1 → Stage 2.

Optional / disabled pre-check leaves today’s default-prompt GAM path unchanged.

---

## Related

- [AutoRAG optimization settings](./experiment_settings.md) — presets and search-space dimensions
- [RAG templates](./rag_templates.md) — current simple template vs planned Graph RAG
- [RAG pattern evaluation](./rag_pattern_evaluation.md) — Stage 2 metrics and judge model
- [RAG pattern inference](./rag_pattern_inference.md) — `pattern.json` generation fields
- [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- [DSPy](https://github.com/stanfordnlp/dspy) — MIPROv2 prompt optimization
