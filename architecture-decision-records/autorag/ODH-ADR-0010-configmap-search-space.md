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

Accept an optional Kubernetes ConfigMap reference as an AutoRAG optimization pipeline input. The ConfigMap defines an advanced search space: embedding and generation models, their endpoint and Secret references, chunking choices, retrieval choices, and generation settings. A single experiment can compare models served directly by vLLM with models accessed through Models as a Service (MaaS).

This extends the preset-based configuration in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md) and the model bindings used by indexing, inference, and evaluation.

## Why

Users need corpus-specific chunking and generation choices, and access to models behind independently authenticated endpoints. A single configuration document groups these settings into a reusable pipeline input. The RFEs listed above request direct vLLM support and advanced search-space control. The advanced-settings RFE includes Dashboard, pipeline, and API access; the pipeline input is an initial increment.

## Goals

* Support custom model, chunking, retrieval, and generation candidates through a validated ConfigMap.
* Carry selected endpoint and Secret bindings through optimization, indexing, and inference.
* Preserve existing preset-based runs.

## Non-Goals

* Model provisioning, subscription management, or credential creation.
* Changes to corpus, database, benchmark, optimization-budget, or metric inputs.
* Provider plugins or the customer-hosted RAG contract from [ODH-ADR-0007](./ODH-ADR-0007-custom-rag-templates.md).

## How

### Pipeline input and precedence

Add `search_space_configmap_name: str = ""` to `documents_rag_optimization_pipeline`. A non-empty value references a ConfigMap in the run's namespace. AutoRAG reads the search-space document from the fixed `data["search-space.yaml"]` key.

An empty ConfigMap name preserves existing behavior. When supplied, the ConfigMap provides model candidates and search-space overrides; `maas_secret_name`, `embedding_models`, and `generation_models` must be empty. Make these legacy inputs optional in the pipeline signature and validate their requirements according to the selected mode. Conflicting inputs fail validation.

The `preset` determines extraction behavior and runtime defaults. Omitted search dimensions inherit preset defaults; supplied candidate lists replace them.

### Configuration

The search-space document uses `apiVersion: autorag.opendatahub.io/v1alpha1` and `kind: AutoRAGSearchSpace`. AutoRAG validates this document; Kubernetes validates the enclosing `v1` ConfigMap.

Example using a MaaS embedding model and both direct vLLM and MaaS generation models. Model ids are illustrative.

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

`models.embedding` and `models.generation` are non-empty lists. Each entry supplies the endpoint's `model_id`, API `provider`, and `secret_name`. Multiple models may share a Secret. The `openai` provider supports direct vLLM and compatible LLMInferenceService endpoints.

Search dimensions use finite, non-empty candidate lists; a one-element list fixes a value. `generation.parameters` applies to all generation models and supports `temperature` and `max_completion_tokens`. Users needing different settings per model submit separate runs.

Chunking and retrieval fields follow [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md), including method-specific fields such as hybrid chunking's `include_metadata`. Publish types, units, bounds, and defaults for each exposed dimension.

### Endpoint and credential binding

Each model's `secret_name` references a Secret in the run's namespace. Both providers use OpenAI-compatible operations, passing `model_id` in the request. The adapter maps generation settings to supported request fields. The provider determines the required, non-empty Secret keys:

| Provider | Required Secret keys | Contract |
| -------- | -------------------- | -------- |
| `maas` | `MAAS_BASE_URL`, `MAAS_API_KEY` | Reuse AutoRAG's existing MaaS Connection format. |
| `openai` | `OPENAI_BASE_URL`, `OPENAI_API_KEY` | New AutoRAG contract for direct OpenAI-compatible inference, including vLLM. |

`OPENAI_BASE_URL` is the complete API base, including any required path and `/v1`. Normalize a legacy `MAAS_BASE_URL`, preserving its gateway path prefix and ensuring a single `/v1` suffix.

For an unauthenticated `openai` endpoint, supply `base_url` on the model entry. Each entry must specify exactly one of `secret_name` or `base_url`; `maas` requires `secret_name`. Missing or inaccessible declared Secrets fail validation.

Within each model-role list, identify candidates by `(provider, secret_name, model_id)`, using the normalized `base_url` in place of `secret_name` for unauthenticated entries. Clients and caches preserve the provider and resolved endpoint binding as part of model identity.

### Alignment with existing OpenShift AI configuration

The [LLMInferenceService scheduler](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/deploy_models_using_distributed_inference_with_llm-d/configuring-llm-scheduler) stores reusable `EndpointPickerConfig` settings in a ConfigMap, referenced through `spec.router.scheduler.config.ref.name` and `.key`. AutoRAG follows this storage pattern with its fixed `search-space.yaml` key. [LLMInferenceServiceConfig](https://access.redhat.com/articles/7149159) provides serving templates through `spec.baseRefs`; AutoRAG consumes the resulting inference endpoint.

Secret handling follows AutoRAG's existing MaaS Connection format. The [Gen AI custom-endpoint fixture](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/k8smocks/testdata/custom_endpoint_models.yaml) demonstrates arbitrary Secret name/key references; AutoRAG uses fixed provider keys to keep model entries concise.

### Resolution and validation

Before document extraction or model pre-selection, read the ConfigMap once, apply defaults, and validate the effective configuration. Persist it with the existing preparation report and MLflow artifacts, recording the ConfigMap name, UID, resource version, payload API version, normalized endpoints, and configuration digest. All downstream steps consume this snapshot; ConfigMap edits affect subsequent runs.

Validation must:

* Check the payload identity, supported fields and types, provider and Secret/URL forms, non-empty candidate lists, and candidate uniqueness within each role.
* Check template/backend constraints and known model limits, including overlap smaller than chunk size and shared generation settings supported by every selected model. Report invalid combinations with actionable field paths.
* Check required Secret keys and probe each model's embedding or generation operation. Endpoint failures during optimization retain existing trial failure semantics.
* Enforce existing search-space bounds and trial budgets, and show the effective search-space size before optimization.

### Evaluation and winning-pattern consumers

Resolve all model calls through their selected bindings, including pre-selection, optimization, model-assisted chunking, graph extraction, and evaluation. Preserve [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md)'s first-generation-model and first-embedding-model evaluator selection, including their endpoint bindings, fixed across trials.

Winning patterns record selected model ids, provider/endpoint bindings, Secret names, and effective search settings. Extend [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md)'s indexing and inference contracts so generated notebooks, starter kits, Dashboard tests, and deployed inference use those bindings. Index reuse accounts for the embedding binding and effective indexing settings. Legacy patterns retain their existing MaaS interpretation.

Indexing and inference use the selected pattern snapshot and resolve credentials at runtime. Deployment into another namespace requires mapping Secret references to authorized Connections there. Compare each Secret's normalized endpoint with the recorded binding; a change requires explicit rebinding and revalidation. Changed embedding behavior requires reindexing. Credential rotation preserves model identity. Model weights remain subject to the endpoint's lifecycle.

## Alternatives

The comparison includes the proposed ConfigMap input. Custom AI Asset selection can be combined with either structured configuration input; it supplies model bindings while chunking, retrieval, and generation settings use the chosen input.

| Option | Advantages | Drawbacks |
| ------ | ---------- | --------- |
| Reference a search-space ConfigMap | Named, reusable configuration shared across runs; supports GitOps and Kubernetes access controls; keeps the pipeline signature small. | Adds resource maintenance, lookup RBAC, ownership, and cleanup. Dashboard creation per run or edit can accumulate many ConfigMaps. Stored payloads require schema compatibility or migration during upgrades; shared edits affect later runs, and reproducibility requires a snapshot. |
| Pass one structured input: `advanced_config: dict` | Groups nested settings and model bindings in one KFP object input; records submitted settings with the run; avoids a separate configuration resource and lookup permissions. New search-space fields can use the same pipeline parameter. | Requires schema validation and Dashboard form support. Older saved configurations and reruns still need compatible parsing or migration after upgrades. Reuse requires a saved configuration mechanism or resubmitting the object; dynamic Secret injection needs pipeline integration. |
| Reuse custom AI Asset endpoints for model selection | Reuses endpoint registration, discovery, and credential references; users can select registered models without entering URLs and Secret details for each run. | Adds dependency on the AI Assets configuration and its schema. Its ConfigMap and Secrets still need lifecycle management. Requires authorized resolution, pipeline credential binding, and a run snapshot; advanced search settings need a separate input. |
| Pass embedding and generation Secret names explicitly | Small, clear interface for one binding per role; reuses pipeline Secret injection; endpoint credentials can rotate independently of pipeline settings. | Requires a defined Secret format and additional inputs for model IDs and search settings. Multiple independently authenticated candidates per role need a richer binding structure. |
| Pass separate inputs such as `chunk_sizes` and `temperatures` | Each setting has its own pipeline parameter, type, and default; straightforward forms for a small set of settings. | Each new dimension extends the pipeline signature and submission interface. Per-model bindings and nested settings are awkward to express as separate parameters. |
| Require all models behind one MaaS Connection | Reuses the existing routing and credential contract; fewer endpoint bindings to configure and maintain. | Requires models to be exposed through the same MaaS Connection; excludes direct vLLM endpoints and comparisons across independently authenticated endpoints. Advanced search settings still need an input. |

For ConfigMap delivery, define whether Dashboard configurations are reusable named resources or generated per run. Generated resources need an ownership and retention policy that covers failed submissions and deleted runs. Upgrades must specify which stored payload versions remain readable and how users migrate unsupported versions; Kubernetes validates the ConfigMap envelope, not AutoRAG's embedded schema.

### Structured versus separate pipeline inputs

Both forms carry the same settings. The difference is the public pipeline interface. For example, these inputs express the same two search dimensions; model bindings and other inputs are omitted:

```python
# One structured pipeline input
advanced_config={
    "chunking": {"chunk_size": [256, 512]},
    "generation": {"parameters": {"temperature": [0.0, 0.2]}},
}

# Two separate pipeline inputs
chunk_sizes=[256, 512]
temperatures=[0.0, 0.2]
```

Passing the configuration as a YAML/JSON string is an encoding variant of the structured option. It carries the same document, but KFP treats it as text and a component must decode it before schema validation. Prefer `dict` where supported; a string adds parsing and escaping without changing the configuration model.

### Dictionary pipeline input

An alternative input is `advanced_config: dict = {}`, carrying the same search-space object as the ConfigMap payload. KFP serializes the object as JSON and represents it as a `STRUCT` parameter. An empty object retains the existing preset-based flow. Apply the same schema, precedence, and validation rules; if both advanced input forms are offered, require users to select one.

An initial component parses the dictionary and outputs the effective settings and Secret names. Downstream tasks pass those names to `kfp.kubernetes.use_secret_as_env()`, which accepts component output parameters. For the two-Secret approach, expose `embedding_secret_name` and `generation_secret_name` as parser outputs and inject each into distinct embedding and generation environment variables. Secret keys and environment-variable mappings are fixed in the pipeline definition for each supported provider.

The parser handles runtime dictionary access; pipeline-authoring code wires its outputs to the consuming tasks. Only Secret references pass between components. Kubernetes injects credential values into the model tasks, allowing the parser and model code to operate without Kubernetes Secret API reads. Submission must authorize the referenced Secrets.

A compilation probe using KFP and kfp-kubernetes 2.16.1 verified the `STRUCT` input, parser outputs, downstream Secret references, and task dependencies. This verifies a fixed pair of Secret bindings; supporting multiple independently authenticated endpoints per model role requires additional injection design. Execution on the target RHOAI pipeline backend remains an integration check.

### Custom AI Asset endpoints

An alternative model-selection flow reuses the Dashboard's custom AI Asset endpoint registration for user-managed OpenAI-compatible endpoints, including direct vLLM deployments. The existing flow stores endpoint and model metadata in the project's `gen-ai-aa-custom-model-endpoints` ConfigMap under `config.yaml`, with credentials referenced by Secret name and key.

1. The user registers an endpoint or selects an existing custom asset for each embedding or generation candidate.
2. At submission, the Dashboard resolves each selection under the submitting user's permissions into its model ID, API base URL, and Secret name/key reference.
3. The pipeline receives the resolved bindings alongside the search settings. Snapshot the selected asset identity and resolved non-secret configuration with the run; subsequent asset edits affect new runs.
4. Consuming components receive the referenced credentials through pipeline Secret injection, authorized for the run independently of the interactive Dashboard session. Winning patterns preserve these bindings for indexing and inference.

This option requires pipeline bindings that accept a base URL and the existing Secret name/key mapping. The Dashboard's custom-endpoint flow currently writes the `api_key` key, while its configuration supports other Secret keys. Preserve that mapping when adapting the fixed provider-key contract above. Verify Secret injection and endpoint reachability from pipeline pods as part of integration. The [custom-endpoint configuration implementation](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/token_k8s_client.go) and [example configuration](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/k8smocks/testdata/custom_endpoint_models.yaml) provide the existing storage contract.

## Security and Privacy Considerations

Authorize the ConfigMap and the Secret references in its snapshot for the submitting user. Limit runtime access to the referenced namespace-local resources and provide credentials only to components that use them. Reuse pipeline Secret injection where possible; dynamic resolution must preserve the same authorization scope.

Keep credentials in Secrets and runtime memory, excluding them from pipeline parameters and artifacts. Reject inline tokens, authentication headers, and credentials embedded in URLs; redact authentication material from logs and errors. Apply project access controls to configuration artifacts, which can reveal endpoints, model names, and prompts. Model requests follow platform egress and data-access policies.

## Risks

* Endpoint capabilities and availability vary. Early validation reduces failures; runtime requests can still exceed context limits or encounter endpoint changes.
* Multiple bindings require coordinated changes across pipeline preparation, optimization, artifact consumers, and Dashboard submission. Release ConfigMap mode once indexing and inference preserve the selected bindings.

## References

* [AutoRAG RAG templates](./ODH-ADR-0003-rag-templates.md)
* [RHOAI Connections API](../operator/ODH-ADR-Operator-0009-connection-api.md)
* [KServe scheduler reference and base-reference API types](https://github.com/kserve/kserve/blob/master/pkg/apis/serving/v1alpha1/llm_inference_service_types.go)
* [Kubernetes component configuration: kube-scheduler](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
* [KFP dictionary input parameters](https://www.kubeflow.org/docs/components/pipelines/user-guides/data-handling/parameters/)
* [KFP Kubernetes Secret injection](https://kfp-kubernetes.readthedocs.io/en/kfp-kubernetes-2.17/_modules/kfp/kubernetes/secret.html)
