# Extra Credit: Centralized Logging With Loki And Promtail

This extra-credit extension adds centralized logging to the ChefMate Kubernetes deployment on Chameleon using `Loki` and `Promtail`.

## Why This Improves Operability

This project ran into real operational issues during Q2 validation:

1. `Mealie` failed when it could not reach its database across nodes.
2. `MLflow` briefly entered `CrashLoopBackOff` and then `OOMKilled` during startup.
3. Old pods often disappeared or were replaced while debugging, which made point-in-time `kubectl logs` collection fragile.

Centralized logging materially improves operability because application logs are shipped off the individual workload and stored in one searchable place.

That gives two concrete benefits:

1. logs remain queryable even after a pod restarts or is replaced
2. logs from multiple namespaces can be queried from one backend instead of jumping through repeated `kubectl logs` commands

## Concrete Operational Win

A realistic example from this project is `MLflow` startup instability.

Without centralized logging, debugging required racing `kubectl logs`, `kubectl logs --previous`, and repeated restarts while the pod changed names and IPs.

With `Loki + Promtail`, we can query all `mlflow` logs centrally by label even after the pod has restarted. That makes it much easier to confirm whether the issue was:

1. application startup delay
2. backend dependency failure
3. resource pressure such as `OOMKilled`

## Artifacts Added

- `ansible/argocd/argocd_add_logging.yml`
- `k8s/logging/loki-values.yaml`
- `k8s/logging/promtail-values.yaml`

## Deployment Steps

From the Trovi/Jupyter control host:

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml argocd/argocd_add_logging.yml
```

Then verify from `node1`:

```bash
kubectl -n chefmate-logging get pods -o wide
kubectl -n chefmate-logging get pvc
kubectl -n chefmate-logging get svc
```

Expected result:

1. a persistent `loki` pod is running on `node1`
2. a `promtail` pod is running on `node1`
3. Loki has a bound PVC

If `loki` fails immediately with a config validation error mentioning `compactor.delete-request-store`, make sure `k8s/logging/loki-values.yaml` includes:

```yaml
loki:
  compactor:
    retention_enabled: true
    delete_request_store: filesystem
```

Then rerun the logging playbook.

## Demo Flow

### 1. Show the logging stack is running

```bash
kubectl -n chefmate-logging get pods -o wide
kubectl -n chefmate-logging get pvc
kubectl -n chefmate-logging get svc
```

### 2. Create some fresh application logs

For example:

```bash
kubectl -n chefmate-platform rollout restart deployment/mlflow
kubectl -n chefmate-platform rollout status deployment/mlflow
```

### 3. Access Loki

If direct node access is unreliable, use an SSH tunnel from the laptop to the current Loki pod IP or service IP.

Example:

```bash
kubectl -n chefmate-logging get pods -o wide
```

Then from the laptop:

```bash
ssh -N -i ~/.ssh/id_rsa_chameleon \
  -L 13100:<LOKI_TARGET_IP>:3100 \
  cc@<FLOATING_IP>
```

### 4. Query logs through Loki

Example queries from the laptop:

```bash
curl -G -s http://localhost:13100/loki/api/v1/query_range \
  --data-urlencode 'query={namespace="chefmate-platform"}' \
  --data-urlencode 'limit=20'

curl -G -s http://localhost:13100/loki/api/v1/query_range \
  --data-urlencode 'query={namespace="chefmate-platform", container="mlflow"}' \
  --data-urlencode 'limit=20'
```

## Short Justification For Submission

We integrated `Loki + Promtail` to add centralized logging, which was not part of the lab assignments. This improved operability by making logs from `Mealie`, `MLflow`, and the rest of the Kubernetes deployment queryable from one durable backend even after pods restarted. A realistic operational win was debugging `MLflow` startup instability: instead of racing `kubectl logs` while pods restarted and changed names, we could query `mlflow` logs centrally by label and confirm the behavior after rollout and restart events.
