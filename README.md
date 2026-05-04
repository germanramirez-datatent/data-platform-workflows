# Data Platform Workflows

Argo Workflows and Kubernetes manifests for orchestrating the shopping center data platform. The workflows ingest raw data, validate payloads, trigger AWS Glue transformations, and run dbt analytics models.

## What Runs Here

The main workflow template is `templates/ingest-all-sources-template.yaml`. It orchestrates:

1. Ingest traffic, weather, holidays, and flights in parallel.
2. Validate each raw payload.
3. Trigger the AWS Glue transform job for each successfully validated source.
4. Run dbt once all curated transformations finish.
5. Print a final status summary.

The scheduled entrypoint is `crons/daily-all-sources.yaml`, which references the workflow template and runs daily at `02:00` in the `Europe/Madrid` timezone.

## Repository Layout

```text
.
|-- crons
|-- minio
|-- pipelines
|-- rbac
|-- simulation-api
`-- templates
```

## Key Manifests

| Path | Purpose |
| --- | --- |
| `templates/ingest-all-sources-template.yaml` | Reusable Argo WorkflowTemplate for the full platform pipeline. |
| `crons/daily-all-sources.yaml` | Daily CronWorkflow that invokes the all-sources template. |
| `pipelines/ingest-all-sources.yaml` | Standalone all-sources workflow for direct submission. |
| `pipelines/ingest-traffic*.yaml` | Earlier traffic-only workflow examples for local and S3-backed runs. |
| `simulation-api/simulation-api.yaml` | Kubernetes Deployment and Service for the local simulation API. |
| `minio/minio.yaml` | Local MinIO deployment, service, and persistent volume claim. |
| `rbac/workflow-rbac.yaml` | ServiceAccount, Role, and RoleBinding used by Argo workflow pods. |
| `pipelines/hello-world.yaml` | Minimal Argo smoke-test workflow. |

## Prerequisites

- Kubernetes cluster with Argo Workflows installed.
- `kubectl` configured for the target cluster.
- `argo` CLI for direct workflow submission.
- Container images built and available to the cluster:
  - `simulation-api:local`
  - `python-ingestor:local`
  - `data-quality:local`
  - `glue-trigger:local`
  - `dbt-runner:local`
- AWS credentials secret for S3, Glue, Athena, and dbt steps.
- OpenSky credentials secret for the flights ingestion step.

For local development, `data-platform-infra/scripts/init-local.sh` installs Argo, MinIO, the simulation API, RBAC, and the daily traffic CronWorkflow into a k3d cluster.

## Required Secrets

The AWS-backed workflows expect:

```bash
kubectl -n argo create secret generic aws-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=<access-key> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<secret-key>
```

Flights ingestion expects:

```bash
kubectl -n argo create secret generic opensky-credentials \
  --from-literal=OPENSKY_CLIENT_ID=<client-id> \
  --from-literal=OPENSKY_CLIENT_SECRET=<client-secret>
```

For local MinIO examples, the bootstrap script creates:

```text
minio-credentials
```

with access key `minioadmin` and secret key `minioadmin`.

## Apply Core Resources

```bash
kubectl apply -f rbac/workflow-rbac.yaml
kubectl apply -f minio/minio.yaml
kubectl apply -f simulation-api/simulation-api.yaml
kubectl apply -f templates/ingest-all-sources-template.yaml
kubectl apply -f crons/daily-all-sources.yaml
```

## Submit Workflows

Submit the reusable template:

```bash
argo submit --from workflowtemplate/ingest-all-sources -n argo --watch
```

Override the ingest date:

```bash
argo submit --from workflowtemplate/ingest-all-sources \
  -n argo \
  -p ingest_date=2026-04-22 \
  --watch
```

Submit the standalone all-sources workflow:

```bash
argo submit pipelines/ingest-all-sources.yaml -n argo --watch
```

Submit a traffic-only local example:

```bash
argo submit pipelines/ingest-traffic-dag.yaml -n argo --watch
```

## Default Parameters

The all-sources template includes default values for development:

| Parameter | Default |
| --- | --- |
| `ingest_date` | `2026-04-22` |
| `project_name` | `data-platform` |
| `environment` | `dev` |
| `raw_bucket` | `data-platform-dev-raw-811430801421` |
| `traffic_expected_records` | `24` |
| `weather_expected_records` | `24` |
| `holidays_expected_records` | `0` |
| `flights_expected_records` | `1` |

## Runtime Flow

Ingestion steps use `python-ingestor:local` and write source JSON to the raw bucket. Each step exposes the raw object key through `/tmp/object_key.txt`, which downstream validation uses.

Validation steps use `data-quality:local`:

- Traffic validates `records` using list count.
- Weather validates `hourly.time` using list count.
- Holidays validates `holidays` using list count.
- Flights validates `states` using greater-than-or-equal mode.

Transform steps use `glue-trigger:local` to start the shared Glue job:

```text
data-platform-dev-transform-to-curated
```

The dbt step uses `dbt-runner:local` and runs:

```bash
dbt run --profiles-dir /root/.dbt && dbt test --profiles-dir /root/.dbt
```

## Local Access

Useful port-forwards:

```bash
kubectl -n argo port-forward deployment/argo-workflows-server 2746:2746
kubectl -n argo port-forward deployment/minio 9000:9000
kubectl -n argo port-forward deployment/minio 9001:9001
```

Local UIs:

- Argo UI: `http://localhost:2746`
- MinIO UI: `http://localhost:9001`

## Related Repositories

- `data-platform-images`: source code and Dockerfiles for the workflow containers.
- `data-platform-infra`: AWS Terraform modules and local bootstrap script.
- `data-platform-dbt`: dbt project executed after curated transformations.
