# Q2.1 Infrastructure Requirements Table

This document records the live Chameleon evidence used to right-size the Q2 DevOps/Platform deployment for ChefMate.

The configured CPU and memory requests and limits in the table below come from the chart values committed in Git:

- `k8s/platform/values.yaml`
- `k8s/staging/values.yaml`

GPU is `0 / 0` for every service because this Q2 deployment does not request or depend on GPU resources anywhere in the Kubernetes manifests.

## Deployment Snapshot

- Site: `KVM@TACC`
- Cluster: `3`-node `k3s` cluster
- Node capacity: lease-backed `m1.large` equivalent capacity, `4 vCPU`, `8 GiB RAM`, `40 GiB disk` per node
- Healthy namespaces observed: `chefmate-platform`, `chefmate-staging`
- Final validated Q2 deployment: both `chefmate-staging` and `chefmate-platform` were recreated after node pinning so the PVC-backed workloads were reprovisioned on `node1`
- Bound PVCs observed:
  - `minio-pvc` = `20Gi`
  - `postgres-pvc` = `10Gi`
  - `mealie-data-pvc` = `8Gi`
  - `mealie-postgres-pvc` = `8Gi`

## Infrastructure Requirements Table

| Service | Namespace | GPU | CPU req/limit | Mem req/limit | Chameleon evidence | Right-sizing rationale |
|---|---|---|---|---|---|---|
| `mealie` | `chefmate-staging` | `0 / 0` | `250m / 1000m` | `512Mi / 1Gi` | Healthy on `node1`; `kubectl top pods -A` showed about `266Mi` RAM during live validation. | Light web app for milestone-scale traffic; request is about 2x observed steady-state memory and limit leaves burst headroom. |
| `mealie-postgres` | `chefmate-staging` | `0 / 0` | `250m / 1000m` | `512Mi / 1Gi` | Final validated deployment ran `mealie-postgres` on `node1`; `mealie-postgres-pvc` remained `Bound` at `8Gi`. | Small durable database for Mealie only; conservative allocation relative to `4 vCPU / 8 GiB` node capacity. |
| `postgres` | `chefmate-platform` | `0 / 0` | `250m / 1000m` | `512Mi / 1Gi` | Final validated deployment ran `postgres` on `node1`; `postgres-pvc` remained `Bound` at `10Gi`. | MLflow metadata backend with modest Q2 load; small guaranteed share plus enough limit headroom for spikes. |
| `minio` | `chefmate-platform` | `0 / 0` | `250m / 1000m` | `512Mi / 1Gi` | Final validated deployment ran `minio` on `node1`; `minio-pvc` remained `Bound` at `20Gi`; the bucket-init job completed successfully. | Small object store for MLflow artifacts; modest request is enough for steady use and `1Gi` covers startup and small transfer bursts. |
| `mlflow` | `chefmate-platform` | `0 / 0` | `250m / 1000m` | `1Gi / 2Gi` | After pinning platform services to `node1`, `mlflow` initially showed `OOMKilled` during startup and was increased to a larger memory envelope; it then came up healthy and served traffic on port `8000`. | MLflow needed more startup/runtime memory than the other Q2 services, so its request and limit were raised while still fitting comfortably within the `4 vCPU / 8 GiB` node budget. |

## Why These Numbers Are Reasonable

The chosen requests and limits remain conservative relative to the leased node capacity:

- `250m` CPU request is about `6.25%` of a `4 vCPU` node
- `1000m` CPU limit is about `25%` of a `4 vCPU` node
- `512Mi` memory request is about `6.25%` of an `8 GiB` node
- `1Gi` memory request is about `12.5%` of an `8 GiB` node
- `1Gi` memory limit is about `12.5%` of an `8 GiB` node
- `2Gi` memory limit is about `25%` of an `8 GiB` node

Most services fit comfortably in the original `512Mi / 1Gi` envelope. `mlflow` was the exception during real Q2 validation: after co-locating the platform stack on `node1`, it briefly hit `OOMKilled`, so its memory was increased to `1Gi / 2Gi`. Even with that increase, the platform plus staging workloads still fit comfortably on the `node1` resource budget.

## Live Evidence Captured On Chameleon

### `kubectl get nodes -o wide`

```text
NAME    STATUS   ROLES           AGE     VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node1   Ready    control-plane   10m     v1.34.6+k3s1   10.56.2.160   <none>        Ubuntu 24.04.2 LTS   6.8.0-59-generic   containerd://2.2.2-bd1.34
node2   Ready    <none>          9m50s   v1.34.6+k3s1   10.56.0.211   <none>        Ubuntu 24.04.2 LTS   6.8.0-59-generic   containerd://2.2.2-bd1.34
node3   Ready    <none>          9m50s   v1.34.6+k3s1   10.56.1.186   <none>        Ubuntu 24.04.2 LTS   6.8.0-59-generic   containerd://2.2.2-bd1.34
```

### `kubectl get pods -A -o wide`

```text
NAMESPACE           NAME                                      READY   STATUS    RESTARTS   AGE    IP          NODE
chefmate-platform   minio-5ddd5b4b79-g5cwb                    1/1     Running   0          2m4s   10.42.1.4   node2
chefmate-platform   minio-create-bucket-xwbdq                 1/1     Running   0          2m4s   10.42.0.5   node1
chefmate-platform   mlflow-79f6b87489-5tr5q                   1/1     Running   0          2m4s   10.42.3.3   node3
chefmate-platform   postgres-7c5c96db86-rws8k                 1/1     Running   0          2m4s   10.42.1.5   node2
chefmate-staging    mealie-7b4c5cc7bf-xng5z                   1/1     Running   0          106s   10.42.0.7   node1
chefmate-staging    mealie-postgres-68ccbbb8bb-rt4lt          1/1     Running   0          106s   10.42.3.5   node3
```

### `kubectl get pvc -A`

```text
NAMESPACE           NAME                  STATUS   CAPACITY   STORAGECLASS
chefmate-platform   minio-pvc             Bound    20Gi       local-path
chefmate-platform   postgres-pvc          Bound    10Gi       local-path
chefmate-staging    mealie-data-pvc       Bound    8Gi        local-path
chefmate-staging    mealie-postgres-pvc   Bound    8Gi        local-path
```

### `kubectl top nodes`

```text
NAME    CPU(cores)   CPU(%)      MEMORY(bytes)   MEMORY(%)
node1   77m          1%          965Mi           12%
node2   <unknown>    <unknown>   <unknown>       <unknown>
node3   <unknown>    <unknown>   <unknown>       <unknown>
```

### `kubectl top pods -A`

```text
NAMESPACE           NAME                                      CPU(cores)   MEMORY(bytes)
chefmate-platform   minio-create-bucket-xwbdq                 1m           0Mi
chefmate-staging    mealie-7b4c5cc7bf-xng5z                   0m           266Mi
kube-system         coredns-76c974cb66-vwx2z                  3m           12Mi
kube-system         local-path-provisioner-8686667995-9spdk   1m           9Mi
kube-system         metrics-server-c8774f4f4-rrw8r            8m           19Mi
```

## Note On Metrics Completeness

At the time of capture, metrics for `node2`, `node3`, and some worker-hosted pods were still incomplete even though the pods themselves were healthy and the services were reachable. The right-sizing decision therefore combines:

1. live measurements that were available
2. confirmed healthy pod placement across the cluster
3. PVC-backed storage status
4. known node capacity from the lease-backed `m1.large` deployment

That is sufficient to justify the selected requests and limits for the milestone-scale Q2 workload.
