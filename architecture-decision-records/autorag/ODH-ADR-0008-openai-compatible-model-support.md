# Open Data Hub - AutoRAG OpenAI-Compatible Model Support

|                |            |
| -------------- | ---------- |
| Date           | 2026-11-17 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAIRFE-3919](https://redhat.atlassian.net/browse/RHAIRFE-3919) |
| Other docs:    | [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) · [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md) · [ODH-ADR-0006](./ODH-ADR-0006-mlflow-integration.md) |

## What

Add direct support for OpenAI-compatible embedding and generation models, including standalone vLLM deployments, through `embedding_secret_names: list` and `generation_secret_names: list`. Preserve selected model bindings across optimization, evaluation, indexing, and inference.

## Why

Users need to consume independently hosted models without requiring MaaS installation, registration, or routing. Explicit Secret lists provide reusable model bindings and keep credentials outside pipeline parameters.

## Non-Goals

Model provisioning, subscription or credential management, changes to other experiment inputs, provider plugins, and the customer-hosted RAG contract in [ODH-ADR-0007](./ODH-ADR-0007-custom-rag-templates.md).

## How

### Pipeline inputs

The pipeline parameter surface is defined in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#pipeline-parameters). This ADR defines resolution of `embedding_secret_names` and `generation_secret_names` into OpenAI-compatible model bindings.

Each Secret list must be non-empty and reference resources in the run's namespace. Preserve list order when resolving embedding and generation candidates.

Example direct-endpoint inputs:

```yaml
embedding_secret_names:
  - embedding-model-connection
generation_secret_names:
  - vllm-chat-connection
```

### OpenAI-compatible model Secrets

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

A shared OpenAI-compatible endpoint has one Secret per selected model. This duplicates connection values but keeps each list entry self-contained. The same Secret may appear in both role lists if the model supports both operations.

### Binding requirements

Each run requires valid embedding and generation bindings supporting the requested operations. Record model IDs, endpoints, and Secret references as run provenance; credential values remain in Secrets and authorized runtime components. The run's identity must have access limited to the referenced Secrets. Secret delivery and validation mechanics belong to the implementation.

### Evaluation and winning patterns

Use selected bindings for all model calls. Preserve [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md)'s first-generation-model and first-embedding-model evaluator selection, fixed across trials.

Winning patterns record selected model IDs, binding type (`maas` or `openai`), endpoints, and Secret names. Extend [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) so indexing, generated notebooks, starter kits, Dashboard tests, and deployed inference consume them. Index reuse accounts for embedding bindings and indexing settings. Existing patterns retain their MaaS interpretation.

Map Secrets to authorized Connections when deploying into another namespace. Compare resolved model IDs and endpoints with the snapshot; changes require rebinding and revalidation. Changed embedding behavior requires reindexing. Credential rotation preserves model identity; model weights follow the endpoint's lifecycle.

## Alternatives

| Option | Advantages | Drawbacks |
| ------ | ---------- | --------- |
| Model Secret lists (selected) | Reusable, explicit bindings; no extra configuration resource. | Requires a defined Secret format and scoped runtime Secret reads. |
| ConfigMap containing model bindings | Named, reusable configuration; GitOps support; flexible endpoint and credential references. | Adds lookup RBAC, schema compatibility, ownership, and cleanup. Dashboard creation per run/edit can accumulate ConfigMaps; shared edits affect later runs. |
| Dictionary of model IDs, URLs, and Secret name/key references | Flexible per-model bindings; reuses arbitrary credential formats and shared endpoint credentials; recorded with the run. | More fields and validation than self-contained Secrets; reuse needs saved configuration. |
| Custom AI Asset endpoints | Reuses Dashboard registration, discovery, and credential references. | Depends on AI Assets schema and resource lifecycle; needs adaptation into pipeline bindings. |
| One embedding and one generation Secret | Simple fixed pair; supports KFP Secret environment injection. | Limits independently authenticated candidates to one per role. |
| Require every model behind one MaaS Connection | One shared connection without duplicated credential values. | Requires direct and independently authenticated endpoints to be exposed through MaaS first. |

ConfigMap delivery would require a snapshot and a retention policy covering failed submissions and deleted runs. Kubernetes validates the ConfigMap envelope; AutoRAG validates model bindings. Saved inputs and reruns need compatibility or migration when binding schemas change.

For custom AI Assets, Dashboard resolves selected models under the submitting user's permissions from `gen-ai-aa-custom-model-endpoints` / `config.yaml`. Its bindings carry model ID, base URL, and Secret name/key (`api_key` for Dashboard-created entries). Integration must map them to the selected Secret contract or extend the pipeline binding schema, preserving asset identity in the run snapshot. See the [implementation](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/token_k8s_client.go) and [example configuration](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/k8smocks/testdata/custom_endpoint_models.yaml).

## Security and Privacy Considerations

Authorize Secret references for the submitting user and the run's ServiceAccount independently of the Dashboard session. The referenced-name allow-list must be enforced before execution. Keep credential values out of parameters, snapshots, logs, and errors; reject credentials embedded in URLs. Protect recorded model bindings with project access controls and apply platform egress policies.

## Risks

* OpenAI-compatible Secret-per-model bindings duplicate credentials for shared endpoints; rotation must update each affected Secret. MaaS uses its shared Connection.
* Endpoint capabilities, availability, and model weights can change during a run.
* Runtime-length Secret resolution and the run ServiceAccount's scoped access require integration validation on the target RHOAI pipeline backend.
* Release the new inputs only when candidate-list resolution and downstream pattern consumers preserve the selected bindings.

## References

* [AutoRAG RAG templates](./ODH-ADR-0003-rag-templates.md)
* [RHOAI Connections API](../operator/ODH-ADR-Operator-0009-connection-api.md)
* [KFP list input parameters](https://www.kubeflow.org/docs/components/pipelines/user-guides/data-handling/parameters/)
* [KFP Kubernetes Secret injection](https://kfp-kubernetes.readthedocs.io/en/kfp-kubernetes-2.17/_modules/kfp/kubernetes/secret.html)
* [Kubernetes RBAC resource names](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources)
