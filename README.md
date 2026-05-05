# Data Platform Workflows

Argo Workflows and Kubernetes manifests for orchestrating the shopping center data platform. The workflows ingest raw data, validate payloads, trigger AWS Glue transformations, and run dbt analytics models.

## What Runs Here

The main workflow template is `base/ingest-all-sources-template.yaml`. It orchestrates:

1. Ingest traffic, weather, holidays, and flights in parallel.
2. Validate each raw payload.
3. Trigger the AWS Glue transform job for each successfully validated source.
4. Run dbt once all curated transformations finish.
5. Print a final status summary.

The scheduled entrypoint is `crons/daily-all-sources.yaml`, which references the workflow template and runs daily at `02:00` in the `Europe/Madrid` timezone.

## Repository Layout

```text
.
|-- base
|-- crons
|-- minio
|-- overlays
|-- pipelines
|-- platform
`-- simulation-api
```

## Key Manifests

| Path | Purpose |
| --- | --- |
| `base/ingest-all-sources-template.yaml` | Shared WorkflowTemplate used by the Kustomize overlays. |
| `base/kustomization.yaml` | Shared Argo base: workflow template, RBAC, and cron workflow. |
| `overlays/local` | Local cluster overlay with `:local` images, MinIO, and local secret refs. |
| `overlays/dev` | AWS dev overlay with ECR images, IRSA, ECR-backed simulation API, and OpenSky `ExternalSecret`. |
| `overlays/prod` | AWS prod overlay with prod buckets, IRSA, ECR-backed simulation API, and OpenSky `ExternalSecret`. |
| `platform/externalsecrets` | Cluster-scoped `ClusterSecretStore` for AWS Parameter Store. |
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
- For `local`: local images built and available to the cluster:
  - `simulation-api:local`
  - `python-ingestor:local`
  - `data-quality:local`
  - `glue-trigger:local`
  - `dbt-runner:local`
- For `dev` and `prod`: ECR images available in `eu-west-1`.
- For `local`: `aws-credentials` Kubernetes Secret for workflow pods.
- For `dev` and `prod`: IRSA configured for the `workflow` ServiceAccount.
- For `dev` and `prod`: External Secrets Operator installed plus access to AWS Parameter Store.

For local development, `data-platform-infra/scripts/init-local.sh` installs Argo, MinIO, the simulation API, RBAC, and the daily traffic CronWorkflow into a k3d cluster.

## Secrets and Credentials

Local workflow pods expect:

```bash
kubectl -n argo create secret generic aws-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=<access-key> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<secret-key>
```

In `dev` and `prod`, the OpenSky Kubernetes Secret is created by `ExternalSecret` from AWS SSM Parameter Store. The expected parameter paths are:

```text
/data-platform/dev/opensky/client-id
/data-platform/dev/opensky/client-secret
/data-platform/prod/opensky/client-id
/data-platform/prod/opensky/client-secret
```

For local MinIO examples, the bootstrap script creates:

```text
minio-credentials
```

with access key `minioadmin` and secret key `minioadmin`.

## Apply Resources

```bash
kubectl apply -k platform/externalsecrets
kubectl apply -k overlays/local
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
```

Apply by environment:

- `local`: `kubectl apply -k overlays/local`
- `dev`: `kubectl apply -k platform/externalsecrets && kubectl apply -k overlays/dev`
- `prod`: `kubectl apply -k platform/externalsecrets && kubectl apply -k overlays/prod`

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

The base template and dev overlay use these defaults:

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

In `base`, `dev`, and `prod`, the workflow uses ECR images and `imagePullPolicy: Always`. The `local` overlay patches those runtime images back to `:local` with `imagePullPolicy: Never`.

Ingestion steps write source JSON to the raw bucket and expose the raw object key through `/tmp/object_key.txt`, which downstream validation uses.

Validation steps use `data-quality:local`:

- Traffic validates `records` using list count.
- Weather validates `hourly.time` using list count.
- Holidays validates `holidays` using list count.
- Flights validates `states` using greater-than-or-equal mode.

Transform steps use the `glue-trigger` container to start the shared Glue job:

```text
data-platform-dev-transform-to-curated
```

The dbt step runs:

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
