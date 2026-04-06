# ChefMate IaC Repository

Infrastructure and deployment repository for the ChefMate course project on Chameleon.

This repo keeps the same high-level shape as the course lab materials so the workflow is familiar, but it is not intended to be a line-for-line copy of the lab. The lab is a DevOps reference, not the project specification.

The real success criteria for this repo come from the course project rubric:

1. run the system on Chameleon
2. use Kubernetes for the 4-person team deployment
3. keep IaC/CaC and Kubernetes manifests in Git
4. persist platform state and artifacts across pod restarts
5. keep secrets out of Git

## Control Host Model

We still use the Trovi/Jupyter environment as the control host, because it is a convenient Chameleon-native place to run Terraform, Ansible, and OpenStack CLI commands.

Use this model:

1. Trovi Jupyter server is the control machine only
2. Terraform and Ansible run from the Jupyter terminal
3. the actual project infrastructure is created on Chameleon by the repo
4. do not use ad hoc single-VM notebook workflows as the deployed project system

## Repo Layout

```text
tf/kvm/
ansible/
  general/
  pre_k8s/
  k8s/
  post_k8s/
  argocd/
k8s/
  platform/
  staging/
  canary/
  production/
workflows/
```

## Q2 Scope

For the current DevOps/Platform milestone, the required deployed pieces are:

1. open-source service: `Mealie`
2. shared platform services: `MLflow`, `MinIO`, `Postgres`
3. durable storage for platform state and artifacts
4. Kubernetes deployment on Chameleon

`canary`, `production`, and `workflows` remain in the repo for continuity, but they are not necessary for the minimum Q2 submission.

## Current Validated Q2 Setup

The deployment path that has already been executed successfully on Chameleon is:

1. site: `KVM@TACC`
2. control host: Trovi/Jupyter under `/work/chefmate-iac`
3. Terraform auth uses `clouds.yaml`, `unset $(set | grep -o "^OS_[A-Za-z0-9_]*")`, `OS_CLIENT_CONFIG_FILE`, and `OS_CLOUD=openstack`
4. compute placement uses a lease-backed `flavor_id` when normal scheduling is full
5. all three nodes are attached to both the private cluster network and `sharednet1` for outbound package access
6. only `node1` receives the floating IP
7. `ansible.cfg` uses `stdout_callback = default` and `callback_result_format = yaml` for current `ansible-core`
8. `chefmate-platform` runs `MLflow`, `MinIO`, and `Postgres` with PVC-backed storage
9. `chefmate-staging` runs `Mealie` with PVC-backed storage

This is the setup the Q2 submission should describe. The lab remains a workflow reference, but the grade is based on what was actually deployed on Chameleon.

## Networking Guidance

The lab often uses `sharednet1` plus a floating IP on `node1` because it is a simple teaching pattern. For this project, treat that as a reference pattern only.

The project rubric and Chameleon guidance matter more:

1. assign a floating IP to only one compute instance and use it as a jump host
2. keep the infrastructure as small as possible
3. expose only the services needed for the milestone demo

If `sharednet1` is available, the current Terraform layout can use it.

If `sharednet1` is exhausted, do not keep forcing the lab assumption. Move to a project-owned routed network design instead:

1. create a private project network and subnet
2. create a router with external gateway `public`
3. place all nodes on that private network
4. assign one floating IP to `node1`
5. use `node1` as the SSH and service entrypoint
6. expose services through ingress or controlled node ports

That design still satisfies the project requirements and aligns with Chameleon networking guidance.

## Canonical Bring-Up Sequence

### 1. Prepare the Trovi/Jupyter control host

```bash
cd /work
git clone <YOUR_REPO_URL> chefmate-iac
cd /work/chefmate-iac
```

### 2. Prepare Terraform auth and environment

```bash
cd /work/chefmate-iac/tf/kvm
export PATH=/work/.local/bin:$PATH
unset $(set | grep -o "^OS_[A-Za-z0-9_]*")
```

Place `clouds.yaml` in `tf/kvm/`.

### 3. Provision infrastructure

```bash
cd /work/chefmate-iac/tf/kvm
cp terraform.tfvars.example terraform.tfvars
# update suffix, key, and any network/site-specific values
terraform init
terraform plan
terraform apply
```

### 4. Configure Ansible

```bash
cd /work/chefmate-iac/ansible
cp ansible.cfg.example ansible.cfg
# update ProxyJump host to the Terraform floating IP
```

### 5. Verify connectivity

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml general/hello_host.yml
```

### 6. Prepare nodes and install Kubernetes

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml pre_k8s/pre_k8s_configure.yml

cd /work/chefmate-iac/ansible/k8s/kubespray
ansible-playbook -i ../inventory/mycluster/hosts.yaml --become --become-user=root ./cluster.yml
```

### 7. Post-install cluster setup

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml post_k8s/post_k8s_configure.yml
```

### 8. Deploy platform and app services

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml argocd/argocd_add_platform.yml
ansible-playbook -i inventory.yml argocd/argocd_add_staging.yml
```

## Q2 Submission Mapping

1. `Q2.1`: use `ansible/post_k8s/RIGHT_SIZING.md` as the source for the infrastructure requirements PDF; the resource values come from `k8s/platform/values.yaml` and `k8s/staging/values.yaml`, and the evidence comes from live `kubectl` output on Chameleon
2. `Q2.2`: upload a `.zip` or `.tgz` of this repo
3. `Q2.3`: record Mealie running in Kubernetes on Chameleon
4. `Q2.4`: record MLflow, MinIO, and durable storage behavior on Chameleon

## Operational Notes

1. Do not commit `clouds.yaml`, `terraform.tfvars`, `ansible.cfg`, or any live secrets
2. Do not paste repository files into notebook cells; keep everything as files under `/work/chefmate-iac`
3. Use the floating IP of `node1` for SSH jump-host access and browser demos
4. If a service is not meant to be public, keep it internal and reach it with port-forwarding instead of broad exposure
5. Keep all Chameleon resource names suffixed with your project ID, e.g. `proj01`
6. For `openstack` CLI commands in Jupyter, prefer `unset $(set | grep -o "^OS_[A-Za-z0-9_]*")`, `export OS_CLIENT_CONFIG_FILE=/work/chefmate-iac/tf/kvm/clouds.yaml`, and `--os-cloud openstack`
7. If Terraform fails with `No valid host was found`, create a short Chameleon lease and use the reserved `flavor_id` in `terraform.tfvars`
8. If workers fail during `apt update`, they do not have outbound package access; all nodes need either `sharednet1` egress or a routed private network
9. If Mealie is reachable through a local browser session via an SSH tunnel or browser running on the control host, that still validates the Chameleon deployment; direct floating-IP access is nicer for the demo, but the key requirement is that the service running on Chameleon is functional
10. If Mealie logs show `psycopg2.OperationalError` against `mealie-postgres` after a redeploy, check pod placement and local-path PVC placement; recreating `chefmate-staging` is the fastest clean fix when the database PVC is pinned to the wrong node

## Current Evidence Source For Right-Sizing

Use these commands on the running cluster to populate `ansible/post_k8s/RIGHT_SIZING.md`:

```bash
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get pvc -A
kubectl top nodes
kubectl top pods -A
```

These are the measurements and deployment facts you should cite in the Q2.1 PDF.
