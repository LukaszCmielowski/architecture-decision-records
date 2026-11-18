# Open Data Hub - AutoRAG Search Space from a ConfigMap

|                |            |
| -------------- | ---------- |
| Date           | 2026-11-17 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAIRFE-3919](https://redhat.atlassian.net/browse/RHAIRFE-3919) · [RHAIRFE-3660](https://redhat.atlassian.net/browse/RHAIRFE-3660) |
| Other docs:    | [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) · [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md) · [ODH-ADR-0006](./ODH-ADR-0006-mlflow-integration.md) |

## What

Accept an optional ConfigMap defining AutoRAG's model candidates, endpoint and Secret bindings, chunking, retrieval, and generation settings. A run can compare direct vLLM endpoints with Models as a Service (MaaS). Selected bindings carry through optimization, evaluation, indexing, and inference; existing preset-based runs remain supported.

## Why

Users need corpus-specific search settings and independently authenticated models beyond the presets in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md). The linked RFEs request direct vLLM support and advanced configuration through Dashboard, pipeline, and API access; this ADR defines the pipeline input.

## Non-Goals

Model provisioning, subscription or credential management, changes to other experiment inputs, provider plugins, and the customer-hosted RAG contract in [ODH-ADR-0007](./ODH-ADR-0007-custom-rag-templates.md).

## How

### Pipeline input and precedence

Add `search_space_configmap_name: str = ""` to `documents_rag_optimization_pipeline`, referencing `data["search-space.yaml"]` in a ConfigMap in the run's namespace. Empty preserves existing behavior. When supplied, `maas_secret_name`, `embedding_models`, and `generation_models` must be empty; make these legacy inputs optional and validate the selected mode.

The `preset` determines extraction behavior and runtime defaults. Omitted search dimensions inherit preset defaults; supplied candidate lists replace them.

### Configuration

AutoRAG validates the versioned document inside the ConfigMap. Model IDs below are illustrative.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: autorag-advanced-search-space
data:
  search-space.yaml: |
    apiVersion: autorag.opendatahub.io/v1alpha1
    kind: AutoRAGSearchSpace
    models:
      embedding:
        - model_id: embedding-model
          provider: maas
          secret_name: maas-connection
      generation:
        - model_id: served-vllm-model
          provider: openai
          secret_name: vllm-connection
        - model_id: maas-chat-model
          provider: maas
          secret_name: maas-connection
    chunking:
      method: [recursive]
      chunk_size: [256, 512, 1024]
      chunk_overlap: [0, 64]
    retrieval:
      search_mode: [vector, hybrid]
      number_of_chunks: [3, 5, 10]
    generation:
      parameters:
        temperature: [0.0, 0.2]
        max_completion_tokens: [512, 1024]
```

`models.embedding` and `models.generation` are non-empty lists of model bindings. Multiple models may share a Secret. The `openai` provider supports direct vLLM and compatible LLMInferenceService endpoints.

Search dimensions use finite, non-empty candidate lists; a one-element list fixes a value. `generation.parameters` applies to all generation models and supports `temperature` and `max_completion_tokens`. Users needing different settings per model submit separate runs.

Chunking and retrieval fields follow [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md), including method-specific fields such as `include_metadata`, with documented types, bounds, units, and defaults.

### Endpoint and credential binding

Each model supplies `model_id`, `provider`, and a namespace-local `secret_name`. Both providers use OpenAI-compatible operations; the adapter maps generation settings to supported fields. Required Secret keys are:

| Provider | Required Secret keys | Contract |
| -------- | -------------------- | -------- |
| `maas` | `MAAS_BASE_URL`, `MAAS_API_KEY` | Reuse AutoRAG's existing MaaS Connection format. |
| `openai` | `OPENAI_BASE_URL`, `OPENAI_API_KEY` | New AutoRAG contract for direct OpenAI-compatible inference, including vLLM. |

`OPENAI_BASE_URL` is the complete API base, including any required path and `/v1`. Normalize a legacy `MAAS_BASE_URL`, preserving its gateway path prefix and ensuring a single `/v1` suffix.

For unauthenticated `openai` endpoints, supply `base_url` instead of `secret_name`; `maas` requires a Secret.

Within each role, identify candidates by `(provider, secret_name, model_id)`, substituting normalized `base_url` for unauthenticated entries. Clients and caches include the resolved endpoint binding in model identity.

### Resolution and validation

Before extraction or model pre-selection, read the ConfigMap once, apply defaults, and validate. Save the effective configuration with the preparation report and MLflow artifacts, including ConfigMap name, UID, resource version, payload version, normalized endpoints, and digest. Downstream steps use this snapshot; edits affect new runs.

Validation must:

* Check schema, unique candidates, and exactly one Secret or URL binding per model.
* Enforce template/backend constraints, model limits, overlap smaller than chunk size, and generation settings supported by every selected model; report errors by field path.
* Check Secret access and required keys, and probe each model's embedding or generation operation. Later endpoint failures retain existing trial failure semantics.
* Enforce search-space bounds and trial budgets, and show the effective search-space size.

### Evaluation and winning-pattern consumers

Use selected bindings for all model calls. Preserve [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md)'s first-generation-model and first-embedding-model evaluator selection, fixed across trials.

Winning patterns record model IDs, provider/endpoint bindings, Secret names, and effective settings. Extend [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) so indexing, generated notebooks, starter kits, Dashboard tests, and deployed inference consume them. Index reuse accounts for embedding bindings and indexing settings. Legacy patterns retain their MaaS interpretation.

Resolve credentials at runtime; map Secrets to authorized Connections when deploying into another namespace. Compare resolved endpoints with the snapshot; changes require rebinding and revalidation. Changed embedding behavior requires reindexing. Credential rotation preserves model identity. Model weights follow the endpoint's lifecycle.

## Alternatives

Configuration delivery and model selection are separate choices and can be combined.

### Configuration delivery

| Option | Advantages | Drawbacks |
| ------ | ---------- | --------- |
| ConfigMap reference (proposed) | Named, reusable configuration; GitOps support; small pipeline signature. | Adds lookup RBAC, resource ownership, maintenance, and cleanup. Dashboard creation per run/edit can accumulate ConfigMaps. Shared edits affect new runs. |
| `advanced_config: dict` | One structured input recorded with the run; no configuration resource or lookup; new fields keep the same parameter. | Needs object validation, Dashboard form support, and a saved-configuration mechanism for reuse. |
| Separate parameters, e.g. `chunk_sizes`, `temperatures` | Individual types and defaults; simple forms for a few settings. | Each dimension extends the pipeline signature; nested model bindings are awkward. |

All forms require input compatibility or migration for saved configurations and reruns after upgrades. Kubernetes validates the ConfigMap envelope; AutoRAG validates its payload. Dashboard-generated ConfigMaps also need retention and cleanup rules for failed submissions and deleted runs.

YAML/JSON text is an encoding variant of the dictionary option: it carries the same document but needs decoding. Prefer `dict` where supported. Separate parameters expose each setting individually rather than grouping them in that document.

#### Dictionary input

`advanced_config: dict = {}` carries the ConfigMap's search-space object as a KFP `STRUCT` parameter. Empty retains legacy behavior; otherwise apply the same schema and precedence. If both forms are offered, accept only one per run.

An initial component outputs effective settings and `embedding_secret_name`/`generation_secret_name`. Downstream tasks use those outputs with `kfp.kubernetes.use_secret_as_env()` and fixed provider-specific key mappings, avoiding Secret API reads in component code.

A KFP/kfp-kubernetes 2.16.1 compilation probe verified this fixed pair of dynamic Secret references. Multiple independently authenticated candidates per role require additional injection design; execution on the target RHOAI backend remains an integration check.

### Model selection

| Option | Advantages | Drawbacks |
| ------ | ---------- | --------- |
| Custom AI Asset endpoints | Reuses Dashboard registration, discovery, and credential references. | Depends on AI Assets schema and resource lifecycle; needs resolution into pipeline bindings. |
| Explicit embedding/generation Secret names | Small interface for one binding per role; reuses pipeline Secret injection. | Needs a defined Secret format and model IDs; multiple bindings per role need a richer structure. |
| One MaaS Connection | Reuses current routing and credentials. | Excludes direct vLLM and independently authenticated endpoints. |

#### Custom AI Asset endpoints

The Dashboard stores custom model metadata in `gen-ai-aa-custom-model-endpoints`, under `config.yaml`, with Secret name/key references. Users register or select assets; at submission, Dashboard resolves each selection under their permissions into a model ID, base URL, and Secret reference passed to the pipeline. Include asset identity in the run snapshot and preserve resolved bindings in winning patterns.

This requires extending pipeline bindings to accept the asset's Secret key (`api_key` for Dashboard-created entries; other keys are supported by the configuration) alongside its base URL. Verify injection and endpoint reachability from pipeline pods. See the [implementation](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/token_k8s_client.go) and [example configuration](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/k8smocks/testdata/custom_endpoint_models.yaml).

## Security and Privacy Considerations

Authorize configuration and Secret references for the submitting user; the run needs its own resource access and endpoint credentials, independent of the Dashboard session. Prefer pipeline Secret injection and limit credentials to consuming components. Keep credential values out of parameters, artifacts, logs, and errors; reject inline credentials and credential-bearing URLs. Protect configuration snapshots with project access controls and apply platform egress policies.

## Risks

* Endpoint capabilities and availability vary. Early validation reduces failures; runtime requests can still exceed context limits or encounter endpoint changes.
* Release the new input only when pipeline preparation, optimization, Dashboard submission, indexing, and inference preserve the selected bindings.

## References

* [AutoRAG RAG templates](./ODH-ADR-0003-rag-templates.md)
* [RHOAI Connections API](../operator/ODH-ADR-Operator-0009-connection-api.md)
* [LLMInferenceService scheduler configuration](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/deploy_models_using_distributed_inference_with_llm-d/configuring-llm-scheduler) — existing ConfigMap reference pattern for `EndpointPickerConfig`.
* [LLMInferenceServiceConfig](https://access.redhat.com/articles/7149159) — serving templates referenced through `spec.baseRefs`.
* [KServe scheduler reference and base-reference API types](https://github.com/kserve/kserve/blob/master/pkg/apis/serving/v1alpha1/llm_inference_service_types.go)
* [Kubernetes component configuration: kube-scheduler](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
* [KFP dictionary input parameters](https://www.kubeflow.org/docs/components/pipelines/user-guides/data-handling/parameters/)
* [KFP Kubernetes Secret injection](https://kfp-kubernetes.readthedocs.io/en/kfp-kubernetes-2.17/_modules/kfp/kubernetes/secret.html)
