# Container / role mapping (initial implementation)

**Scope:** Role → container → build source → K8s manifest. Roles: Training, Serving, Data, DevOps/Platform. Contract: agreed JSON I/O samples. `TBD` where integration is pending. `task.md` Q2.2: IaC/K8s deliverables.

---

## DevOps / IaC

No Dockerfile for provisioning. Sources: Terraform, Ansible, Helm values, manifests. “Compose” column includes IaC paths. Images: upstream unless stated.

---

## Master table

| Course role | Container / workload | Dockerfile, Compose, or equivalent | Equivalent Kubernetes / deployment |
|-------------|------------------------|--------------------------------------|-------------------------------------|
| **DevOps/Platform** | `mealie` | Upstream image `ghcr.io/mealie-recipes/mealie` ([project](https://github.com/mealie-recipes/mealie)) — no ChefMate Dockerfile | [`k8s/staging/templates/mealie.yaml`](../../k8s/staging/templates/mealie.yaml), [`k8s/staging/values.yaml`](../../k8s/staging/values.yaml) |
| **DevOps/Platform** | `mealie-postgres` | Upstream `postgres:17` ([docker-library/postgres](https://github.com/docker-library/postgres)) | Same as above (`mealie` chart) |
| **DevOps/Platform** | `mlflow` | Upstream `ghcr.io/mlflow/mlflow` ([MLflow](https://github.com/mlflow/mlflow)) | [`k8s/platform/templates/mlflow.yaml`](../../k8s/platform/templates/mlflow.yaml), [`k8s/platform/values.yaml`](../../k8s/platform/values.yaml) |
| **DevOps/Platform** | `postgres` (platform DB) | Upstream `postgres:17` | [`k8s/platform/templates/postgres.yaml`](../../k8s/platform/templates/postgres.yaml), [`k8s/platform/values.yaml`](../../k8s/platform/values.yaml) |
| **DevOps/Platform** | `minio` | Upstream `minio/minio` ([MinIO](https://github.com/minio/minio)) | [`k8s/platform/templates/minio.yaml`](../../k8s/platform/templates/minio.yaml), [`k8s/platform/values.yaml`](../../k8s/platform/values.yaml) |
| **DevOps/Platform** | `mc` (bucket init Job) | Upstream `minio/mc` | Same [`k8s/platform/templates/minio.yaml`](../../k8s/platform/templates/minio.yaml) (Job `minio-create-bucket`) |
| **DevOps/Platform** | Cluster VMs, network, k3s | [`tf/kvm/`](../../tf/kvm/) (`main.tf`, `variables.tf`, …) | N/A (cloud layer); cluster is target for manifests below |
| **DevOps/Platform** | Kubernetes bootstrap & config | [`ansible/inventory.yml`](../../ansible/inventory.yml), [`ansible/post_k8s/`](../../ansible/post_k8s/) | Cluster hosts workloads in `chefmate-staging`, `chefmate-platform`, … |
| **DevOps/Platform** | GitOps (Argo CD applications) | [`ansible/argocd/`](../../ansible/argocd/) (`argocd_add_staging.yml`, `argocd_add_platform.yml`, …) | Helm charts from repo via Argo CD |
| **DevOps/Platform** | `loki` (optional observability) | Grafana Helm chart values — no local Dockerfile | [`k8s/logging/loki-values.yaml`](../../k8s/logging/loki-values.yaml), [`ansible/argocd/argocd_add_logging.yml`](../../ansible/argocd/argocd_add_logging.yml) |
| **DevOps/Platform** | `promtail` (optional) | Grafana Helm chart values | [`k8s/logging/promtail-values.yaml`](../../k8s/logging/promtail-values.yaml), [`ansible/argocd/argocd_add_logging.yml`](../../ansible/argocd/argocd_add_logging.yml) |
| **Training** | Training job container(s) | **TBD** — Training repo `Dockerfile` | **TBD** — MLflow: [`k8s/platform/templates/mlflow.yaml`](../../k8s/platform/templates/mlflow.yaml); optional `Job`/`Workflow` |
| **Data** | Data pipeline / ingest / batch jobs | **TBD** — Data repo Dockerfile / Compose | **TBD** — [`k8s/platform/templates/minio.yaml`](../../k8s/platform/templates/minio.yaml) or future charts |
| **Serving** | `triton_server` | [`docker-compose-triton.yaml`](https://github.com/HivanshD/serving/blob/main/docker-compose-triton.yaml) · [`Dockerfile.triton`](https://github.com/HivanshD/serving/blob/main/Dockerfile.triton) | **TBD** — no Triton chart in repo |
| **Serving** | `triton_client` | Same Compose · [`Dockerfile.triton_client`](https://github.com/HivanshD/serving/blob/main/Dockerfile.triton_client) | **TBD** |
| **Serving** | `jupyter` (Triton stack) | [`docker-compose-triton.yaml`](https://github.com/HivanshD/serving/blob/main/docker-compose-triton.yaml) · upstream `jupyter/base-notebook` | **TBD** |
| **Serving** | `fastapi_server` (ONNX path) | [`docker-compose-fastapi.yaml`](https://github.com/HivanshD/serving/blob/main/docker-compose-fastapi.yaml) · [`fastapi_onnx/Dockerfile`](https://github.com/HivanshD/serving/blob/main/fastapi_onnx/Dockerfile) | **TBD** |
| **Serving** | `jupyter` (FastAPI stack) | [`docker-compose-fastapi.yaml`](https://github.com/HivanshD/serving/blob/main/docker-compose-fastapi.yaml) · upstream `jupyter/base-notebook` | **TBD** |
| **Serving** | FastAPI PyTorch variant (build context) | [`fastapi_pt/Dockerfile`](https://github.com/HivanshD/serving/blob/main/fastapi_pt/Dockerfile) (no root Compose; optional `docker build`) | **TBD** |

---

## Serving: [github.com/HivanshD/serving](https://github.com/HivanshD/serving)

| Compose file | Services | Build definitions |
|--------------|----------|---------------------|
| [docker-compose-triton.yaml](https://github.com/HivanshD/serving/blob/main/docker-compose-triton.yaml) | `triton_server`, `triton_client`, `jupyter` | [Dockerfile.triton](https://github.com/HivanshD/serving/blob/main/Dockerfile.triton), [Dockerfile.triton_client](https://github.com/HivanshD/serving/blob/main/Dockerfile.triton_client), GPU in Compose |
| [docker-compose-fastapi.yaml](https://github.com/HivanshD/serving/blob/main/docker-compose-fastapi.yaml) | `fastapi_server`, `jupyter` | [fastapi_onnx/Dockerfile](https://github.com/HivanshD/serving/blob/main/fastapi_onnx/Dockerfile) |

Serving workloads: not in ChefMate Helm yet (`TBD`). Update `k8s/` when charts + Argo CD apps land.

---

## Updates

- Serving: update K8s column after `k8s/serving/` (or equivalent) + `ansible/argocd/` playbook.
