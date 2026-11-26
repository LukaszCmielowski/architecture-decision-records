# Open Data Hub - AutoML Data Connect Hub Connections

|                |            |
| -------------- | ---------- |
| Date           | 2026-11-25 |
| Scope          | AutoML Component |
| Status         | Proposed |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | [AutoML architecture](./ODH-ADR-0001-automl.md) · [AutoML experiment settings](./ODH-ADR-0002-experiment-settings.md) · [Data Connect Hub architecture](../data-connect-hub/ODH-ADR-0001-data-connect-hub.md) |

## Decision

Support saved Data Connect Hub (DCH) connections for relational tables and S3 objects in the managed AutoML classification, regression, and time-series pipelines. AutoML snapshots the selected data and trains from that snapshot in the same run.

## Why

Users can train from a relational table without manually exporting a file to S3. The snapshot gives training and refit tasks a stable input if the source changes during the run.

## Run inputs

The DCH extension adds three pipeline inputs. S3 through DCH adds only `train_data_connection_id` and reuses the existing file-key inputs.

| Data source | Required training inputs | Optional external test inputs |
| --- | --- | --- |
| Direct S3 with a Secret (current) | `train_data_secret_name`, `train_data_bucket_name`, `train_data_file_key` | `test_data_bucket_name` and `test_data_file_key` using the training Secret. |
| S3 through DCH | `train_data_connection_id`, `train_data_file_key` | `test_data_file_key` in the bucket saved on the training connection. |
| Relational database through DCH | `train_data_connection_id`, `train_data_table` | `test_data_table` on the training connection. |

File keys identify exact S3 objects. Table names use `schema.table_name`. Omit external test inputs to use the existing split behavior. Existing OpenShift AI S3 Connections can use the DCH path once they are available in DCH. [DCH S3 connector](https://github.com/opendatahub-io/data-connect-hub/blob/6d58bc9975a431c8f8a9040e610520d20c980f36/docs/user-guide/connectors.md)

Omitting `train_data_connection_id` selects the existing direct S3 path. Make the training Secret and bucket optional in the pipeline definition, then require them during validation for direct S3 runs. For DCH runs, reject run-supplied Secrets and buckets, missing or conflicting selectors, and selectors that mismatch the connection type. S3 connections through DCH require `AWS_S3_BUCKET` on the saved connection. The Dashboard and API use the same parameters. [DCH bucket requirement](https://github.com/opendatahub-io/data-connect-hub/blob/main/connectors/s3/src/connector.rs#L83-L100)

## Dependency on AI Pipelines

DCH-backed AutoML runs require the AI Pipelines integration that supplies the trusted DCH gateway address and CA bundle to the workload pod that reads the dataset. Enable DCH as an AutoML data source only on pipeline servers with this integration.

The task must be able to resolve and reach the gateway over HTTPS for connection lookup and object downloads, and over Flight SQL gRPC for table reads. A missing gateway setting or TLS verification failure stops the DCH read before the task sends its token.

## Execution and artifacts

1. Validate the inputs. The task uses the run namespace as the DCH tenant and its ServiceAccount token to access the saved connection.
2. Reuse the DCH table export component for Flight SQL and stream S3 objects through the DCH SDK. Publish each complete result as a KFP Dataset artifact in the AI Pipelines artifact store. Disable cross-run caching of DCH reads so each run captures current data and checks access again. [DCH SDK binary download example](https://github.com/opendatahub-io/data-connect-hub/blob/6d58bc9975a431c8f8a9040e610520d20c980f36/sdk/python/examples/binary_download.py)
3. Pass the snapshots to the existing AutoML loader for schema validation and training. Export tables in a format the loader supports and preserve the original format of S3 objects. Record the connection ID, table or object key, and snapshot artifact reference for each dataset. [AutoML input formats](./ODH-ADR-0002-experiment-settings.md)

The pipeline stages snapshots on task disk. For model selection, it subsamples the training data only when it exceeds the size limit defined by the selected `preset`. [AutoML preset support](./ODH-ADR-0002-experiment-settings.md#preset-support)

## Security and privacy

The task sends its ServiceAccount token only to the platform-configured DCH gateway over verified TLS. DCH must accept the token audience and authorize the task identity for connection lookup and dataset reads in the run's namespace. The Dashboard uses the submitter's authorization to list Connections. Runs submitted through the Dashboard or API must follow the platform's run authorization policy, and DCH enforces the workload's access at execution time.

DCH uses the saved connection's credentials to read the database or S3 bucket. Source credentials and ServiceAccount tokens remain outside run parameters, reports, logs, and artifacts. The pipeline artifact store's access and retention rules apply to all snapshots.

## Alternatives

| Approach | Trade-off |
| --- | --- |
| Run the standalone DCH export and submit its snapshot URI to AutoML | Useful before the managed integration lands. Users manage two runs and configure read access to the intermediate object. |
| Mount a DCH-exported Secret and connect directly to each database | Gives AutoML direct access to credentials and adds connector drivers, TLS settings, and authentication behavior to maintain. |
| Stream rows directly into training | Avoids staging a file, but couples loaders and training to DCH availability and makes a stable input across pipeline stages harder to preserve. |

## Verification

* Submit a DCH-backed run from a project with the integration enabled. Verify gateway configuration and TLS trust in the export task pod, successful upload of the Dataset artifact, and successful training with the expected target column.
* Verify tabular and time-series training with a relational connection ID and an S3 connection ID, using formats supported by each loader. Cover an external test dataset on the training connection and a run using the default split.
* Verify that model selection subsamples training data above the preset limit and uses all training data at or below the limit.
* Verify actionable failures for a missing gateway, rejected token or tenant, unauthorized connection, an S3 connection without a saved bucket, selector/connection mismatch, missing object, and an invalid table identifier. Confirm that a failed read leaves no partial dataset for training.
* Run the existing direct S3 path on a pipeline server without DCH configuration.

## Open questions

* Which AI Pipelines configuration source and injection mechanism will publish the gateway and CA configuration to task pods?
