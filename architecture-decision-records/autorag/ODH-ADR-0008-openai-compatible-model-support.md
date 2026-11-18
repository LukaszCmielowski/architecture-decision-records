# Open Data Hub - AutoRAG OpenAI-Compatible Model Support

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

Support Models as a Service (MaaS) through `maas_secret_name` and model-ID lists, and additional OpenAI-compatible models through `embedding_secret_names: list` and `generation_secret_names: list`. Accept search-space overrides through `search_space_config: dict`. A run can compare candidates from both sources; selected bindings carry through optimization, evaluation, indexing, and inference.

## Why

Users need independently authenticated models and corpus-specific settings beyond the presets in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md). Explicit Secret lists keep model connections reusable and credentials outside pipeline parameters; a dictionary groups advanced settings without another Kubernetes resource.

AutoRAG must support OpenAI-compatible embedding and generation endpoints directly, including standalone vLLM deployments, without requiring MaaS installation, registration, or routing. A run using only these endpoints requires neither `maas_secret_name` nor MaaS model-ID inputs.

## Non-Goals

Model provisioning, subscription or credential management, changes to other experiment inputs, provider plugins, and the customer-hosted RAG contract in [ODH-ADR-0007](./ODH-ADR-0007-custom-rag-templates.md).

## How

### Pipeline inputs

The parameter surface is defined in [ODH-ADR-0002 — Pipeline parameters](./ODH-ADR-0002-experiment-settings.md#pipeline-parameters). This ADR defines how `maas_secret_name` with model-ID lists and `embedding_secret_names` / `generation_secret_names` resolve into model bindings. The `search_space_config` schema, defaults, precedence, and validation are owned by [ODH-ADR-0002 — Search-space configuration](./ODH-ADR-0002-experiment-settings.md#search-space-configuration).

Secret names reference resources in the run's namespace. MaaS model IDs require `maas_secret_name`; a supplied MaaS Connection requires at least one selected MaaS model. Each role's candidates are its MaaS model IDs followed by its explicit Secret bindings, preserving list order. Require at least one embedding and one generation candidate across the combined sources. MaaS-only, OpenAI-compatible-only, and mixed runs are supported.

Example model inputs for a mixed run; combine these with `search_space_config` from ADR 2 as needed:

```yaml
maas_secret_name: maas-connection
embedding_models:
  - maas-embedding-model
generation_models:
  - maas-chat-model
embedding_secret_names:
  - embedding-model-connection
generation_secret_names:
  - vllm-chat-connection
```

### MaaS and OpenAI-compatible Secret contracts

The MaaS Connection uses `MAAS_BASE_URL` and `MAAS_API_KEY`; `embedding_models` and `generation_models` select its model IDs. Normalize the API base while preserving the gateway path and ensuring a single `/v1` suffix.

Each Secret in `embedding_secret_names` or `generation_secret_names` supplies one OpenAI-compatible model binding:

| Key | Requirement |
| --- | ----------- |
| `OPENAI_BASE_URL` | Complete API base URL, preserving gateway paths and including `/v1` where required. |
| `OPENAI_MODEL` | Model ID sent in embedding or chat-completion requests. |
| `OPENAI_API_KEY` | Required for authenticated endpoints; omit for unauthenticated endpoints. |

For example, a direct vLLM model uses:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vllm-chat-connection
type: Opaque
stringData:
  OPENAI_BASE_URL: https://vllm.example.com/v1
  OPENAI_MODEL: served-chat-model
  OPENAI_API_KEY: <endpoint-token>
```

A shared OpenAI-compatible endpoint has one Secret per selected model. This duplicates connection values but keeps each list entry self-contained. The same Secret may appear in both role lists if the model supports both operations. MaaS models share their MaaS Connection.

### Resolution and validation

Before extraction or pre-selection, resolve model bindings and apply ADR 2's search-space validation. Check:

* At least one candidate per role across both sources, unique bindings within each role, and required Secret keys.
* Each model's limits, embedding or generation operation, and endpoint reachability.

Model-consuming components read the named Secrets using the run's ServiceAccount, with namespace-local `get` access restricted to the referenced names. Credential values stay inside consuming components. This supports runtime-length candidate lists without compiling a fixed number of Secret environment bindings; verify the access and resolution path on the target RHOAI pipeline backend.

Add ordered model IDs, normalized endpoints, and Secret names to the run snapshot defined in ADR 2. Downstream steps use these bindings and resolve credentials at runtime. Later endpoint failures retain existing trial failure semantics. Identify candidates by Secret name and model ID, including the resolved endpoint in client and cache identity.

### Evaluation and winning patterns

Use selected bindings for all model calls. Preserve [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md)'s first-generation-model and first-embedding-model evaluator selection, fixed across trials.

Winning patterns record selected model IDs, binding type (`maas` or `openai`), endpoints, Secret names, and effective settings. Extend [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) so indexing, generated notebooks, starter kits, Dashboard tests, and deployed inference consume them. Index reuse accounts for embedding bindings and indexing settings. Existing patterns retain their MaaS interpretation.

Map Secrets to authorized Connections when deploying into another namespace. Compare resolved model IDs and endpoints with the snapshot; changes require rebinding and revalidation. Changed embedding behavior requires reindexing. Credential rotation preserves model identity; model weights follow the endpoint's lifecycle.

## Alternatives

### Configuration delivery

| Option | Advantages | Drawbacks |
| ------ | ---------- | --------- |
| MaaS inputs and OpenAI-compatible Secret lists plus `search_space_config: dict` (selected) | Supports MaaS and direct endpoints together; one structured settings input; no extra configuration resource. | Requires both Secret contracts, scoped runtime Secret reads, and object validation. |
| Search-space ConfigMap reference | Named, reusable configuration; GitOps support; small pipeline signature. | Adds lookup RBAC, ownership, maintenance, and cleanup. Dashboard creation per run/edit can accumulate ConfigMaps; shared edits affect later runs. |
| One `advanced_config: dict` containing models and settings | Flexible per-model endpoint, provider, and Secret mappings in one object. | Larger schema and Dashboard form; combines connection definitions with search settings. |
| Separate parameters, e.g. `chunk_sizes`, `temperatures` | Individual types and defaults; simple forms for a few settings. | Each dimension extends the pipeline signature; nested settings are awkward. |

All forms require compatibility or migration for saved configurations and reruns after upgrades. A ConfigMap alternative would read `search-space.yaml` via `search_space_configmap_name`, snapshot the resolved document, and define retention for generated resources, failed submissions, and deleted runs. Kubernetes validates the envelope; AutoRAG validates the payload. [LLMInferenceService scheduler configuration](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/deploy_models_using_distributed_inference_with_llm-d/configuring-llm-scheduler) provides an existing ConfigMap reference pattern.

YAML/JSON text is an encoding variant of a dictionary input and requires decoding before validation.

### Model bindings

| Option | Advantages | Drawbacks |
| ------ | ---------- | --------- |
| Custom AI Asset endpoints | Reuses Dashboard registration, discovery, and credential references. | Depends on AI Assets schema and resource lifecycle; needs adaptation into pipeline bindings. |
| One embedding and one generation Secret | Simple fixed pair; supports KFP Secret environment injection. | Limits independently authenticated candidates to one per role. A compilation probe for this pair does not validate arbitrary Secret lists. |
| Require every model behind one MaaS Connection | One shared connection without duplicated credential values. | Requires direct and independently authenticated endpoints to be exposed through MaaS first. |
| Model entries with explicit URL and Secret name/key | Reuses arbitrary credential formats and shared gateway credentials. | More per-model fields and validation than a self-contained Secret. |

For custom AI Assets, Dashboard resolves selected models under the submitting user's permissions from `gen-ai-aa-custom-model-endpoints` / `config.yaml`. Its bindings carry model ID, base URL, and Secret name/key (`api_key` for Dashboard-created entries). Integration must map them to the selected Secret contract or extend the pipeline binding schema, preserving asset identity in the run snapshot. See the [implementation](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/token_k8s_client.go) and [example configuration](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/k8smocks/testdata/custom_endpoint_models.yaml).

## Security and Privacy Considerations

Authorize Secret references for the submitting user and the run's ServiceAccount independently of the Dashboard session. The referenced-name allow-list must be enforced before execution. Keep credential values out of parameters, snapshots, logs, and errors; reject credentials embedded in URLs. Protect configuration snapshots with project access controls and apply platform egress policies.

## Risks

* OpenAI-compatible Secret-per-model bindings duplicate credentials for shared endpoints; rotation must update each affected Secret. MaaS uses its shared Connection.
* Endpoint capabilities, availability, and model weights can change during a run.
* Release the new inputs only when candidate-list resolution and downstream pattern consumers preserve the selected bindings.

## References

* [AutoRAG RAG templates](./ODH-ADR-0003-rag-templates.md)
* [RHOAI Connections API](../operator/ODH-ADR-Operator-0009-connection-api.md)
* [KFP dictionary and list input parameters](https://www.kubeflow.org/docs/components/pipelines/user-guides/data-handling/parameters/)
* [KFP Kubernetes Secret injection](https://kfp-kubernetes.readthedocs.io/en/kfp-kubernetes-2.17/_modules/kfp/kubernetes/secret.html)
* [Kubernetes RBAC resource names](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources)
