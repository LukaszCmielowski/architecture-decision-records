# Open Data Hub - AutoRAG OpenAI-Compatible Model Support

|                |            |
| -------------- | ---------- |
| Date           | 2026-11-17 |
| Scope          | AutoRAG Component |
| Status         | Proposed |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAIRFE-3919](https://redhat.atlassian.net/browse/RHAIRFE-3919) |
| Other docs:    | [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) · [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md) · [ODH-ADR-0006](./ODH-ADR-0006-mlflow-integration.md) |

## What

Add direct support for one OpenAI-compatible embedding model and one generation model, including standalone vLLM deployments, through `embedding_model_secret_name: str` and `generation_model_secret_name: str`.

## Why

Users need to consume independently hosted models without requiring MaaS installation, registration, or routing.

## Non-Goals

Model provisioning, subscription or credential management, changes to other experiment inputs, provider plugins, and the customer-hosted RAG contract in [ODH-ADR-0007](./ODH-ADR-0007-custom-rag-templates.md).

## How

### Pipeline inputs

The pipeline parameter surface is defined in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#pipeline-parameters). Direct OpenAI-compatible access requires both Secret names, referencing resources in the run's namespace. Each Secret identifies one model supporting its assigned operation, fixed throughout the run.

**Use MaaS for multiple embedding or generation model candidates in one experiment's search space**, through `maas_secret_name`, `embedding_models`, and `generation_models`.

Example direct-endpoint inputs:

```yaml
embedding_model_secret_name: embedding-model-connection
generation_model_secret_name: vllm-chat-connection
```

### OpenAI-compatible model Secrets

Each referenced Secret supplies one OpenAI-compatible model binding:

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

### Evaluation and winning patterns

Use the selected bindings for all model calls, including evaluation, fixed across trials ([ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md)).

Runs and winning patterns record selected model IDs, binding type (`maas` or `openai`), endpoints, and Secret names. Consumers defined in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md)—indexing, notebooks, starter kits, Dashboard tests, and deployed inference—preserve these bindings. Index reuse accounts for embedding bindings and indexing settings. Existing patterns retain their MaaS interpretation.

Model identity includes the endpoint binding; model ID alone is insufficient for artifact consumption and index reuse.

Changing the embedding model or its behavior requires reindexing.

## Alternatives

| Option | Advantages | Drawbacks |
| ------ | ---------- | --------- |
| One embedding and one generation Secret (selected) | Fixed, explicit bindings; supports KFP Secret injection; no extra configuration resource. | One model per role; multi-model search uses MaaS. |
| ConfigMap containing model bindings | Named, reusable configuration; GitOps support; flexible endpoint and credential references. | Adds lookup RBAC, schema compatibility, ownership, snapshots, retention, and cleanup. Dashboard creation per run/edit can accumulate ConfigMaps; shared edits affect later runs. |
| Dictionary of model IDs, URLs, and Secret name/key references | Flexible per-model bindings; reuses arbitrary credential formats and shared endpoint credentials; recorded with the run. | More fields and validation than self-contained Secrets; reuse needs saved configuration. |
| [Custom AI Asset endpoints](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/bff/internal/integrations/kubernetes/k8smocks/testdata/custom_endpoint_models.yaml) | Reuses Dashboard registration, discovery, and credential references. | Tech Preview with an unclear GA path. Depends on AI Assets schema and resource lifecycle; Secret name/key mappings need adaptation into pipeline bindings. |
| Model Secret lists | Supports multiple independently authenticated models per role. | More selection, validation, and credential-injection complexity than a fixed pair. |
| Require every model behind one MaaS Connection | One shared connection without duplicated credential values. | Requires direct and independently authenticated endpoints to be exposed through MaaS first. |

## Security and Privacy Considerations

Authorize the submitting user and run identity for only the referenced Secrets, independently of the Dashboard session. Keep credential values in Secrets and authorized runtime components, excluding parameters, snapshots, logs, and errors. Reject credentials embedded in URLs. Protect recorded bindings with project access controls and apply platform egress policies.

## Risks

* Separate model Secrets can duplicate shared-endpoint credentials; rotation must update each affected Secret.
* Endpoint capabilities, availability, and model weights can change during a run.
* Secret injection and run authorization require integration validation on the target RHOAI pipeline backend.

## References

* [AutoRAG RAG templates](./ODH-ADR-0003-rag-templates.md)
* [RHOAI Connections API](../operator/ODH-ADR-Operator-0009-connection-api.md)
* [KFP Kubernetes Secret injection](https://kfp-kubernetes.readthedocs.io/en/kfp-kubernetes-2.17/_modules/kfp/kubernetes/secret.html)
* [Kubernetes RBAC resource names](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources)
