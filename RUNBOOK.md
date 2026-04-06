# Q2 DevOps Runbook

This runbook is the shortest path to finishing the `Q2 DevOps/Platform` milestone for ChefMate on Chameleon.

It uses the course lab as a DevOps reference for tooling and workflow, but the deployment decisions here are driven by the real project rubric, not by reproducing the lab exactly.

## Goal

Produce these four submission artifacts:

1. `Q2.1` PDF: infrastructure requirements table with real Chameleon evidence
2. `Q2.2` repo archive: `.zip` or `.tgz` of this repo
3. `Q2.3` video: Mealie running in Kubernetes on Chameleon
4. `Q2.4` video: shared platform services running in Kubernetes on Chameleon with persistence

## Current Validated Setup

This is the deployment path that has already been executed successfully and should anchor the submission narrative:

1. Trovi/Jupyter is the control host and the repo lives under `/work/chefmate-iac`
2. Terraform auth uses `clouds.yaml`, `unset $(set | grep -o "^OS_[A-Za-z0-9_]*")`, `OS_CLIENT_CONFIG_FILE`, and `OS_CLOUD=openstack`
3. instance placement uses a Chameleon lease-backed `flavor_id` when normal scheduling fails
4. all three nodes have both a private network interface and a `sharednet1` interface for outbound package access
5. only `node1` has the floating IP and acts as the jump host and service entrypoint
6. Ansible uses `stdout_callback = default` and `callback_result_format = yaml`
7. the `chefmate-platform` namespace is healthy with `MLflow`, `MinIO`, and `Postgres`
8. the `chefmate-staging` namespace is the Mealie deployment used for `Q2.3`

If Mealie is currently reachable from a local browser session, that is already strong evidence that the application itself is running on Chameleon. In that case, any remaining issue is likely public exposure or service-routing polish, not a failed deployment.

## Governing Rules For This Repo

Use these rules when making implementation choices:

1. the project spec is the source of truth
2. the lab is a reference pattern for DevOps workflow and structure
3. all real work must run on Chameleon, not only locally
4. Kubernetes is required for the 4-person team deployment
5. platform state and artifacts must persist across pod restarts
6. no secrets belong in Git
7. use only one floating IP unless you have a compelling reason otherwise

## Execution Context

This repo assumes you are using a Chameleon Trovi/Jupyter server as the control host.

That means:

1. Jupyter/Trovi is where you run Terraform, Ansible, and OpenStack CLI commands
2. the Jupyter server itself is not the deployed system
3. do not use notebook-only single-VM flows as the actual Q2 deployment
4. do not paste repo contents into notebook cells; keep the repo as files under `/work`

All commands below assume the repo is available at:

```bash
cd /work/chefmate-iac
```

## What Matters For Q2

The minimum required repo paths for the Q2 DevOps submission are:

- `tf/kvm/`
- `ansible/`
- `k8s/platform/`
- `k8s/staging/`
- `ansible/post_k8s/RIGHT_SIZING.md`

`canary`, `production`, and `workflows` can remain in the repo, but they are not required for the minimum Q2 milestone demo.

## 0. Start The Trovi Control Environment

From Trovi or the course materials:

1. launch the Jupyter environment on Chameleon
2. open JupyterLab
3. if the course intro notebook is present, run the cells that choose the project/site and refresh the keypair
4. open a `Terminal` in JupyterLab

The setup cells are conceptually:

```python
from chi import server, context

context.version = "1.0"
context.choose_project()
context.choose_site(default="KVM@TACC")
server.update_keypair()
```

Important distinction:

1. Jupyter URL is only for the notebook and terminal environment
2. the project services run on the infrastructure created by Terraform
3. the floating IP of `node1` is the browser and SSH entrypoint for the deployed system

## 1. Prepare The Control Host

In the Jupyter terminal:

```bash
cd /work
pwd
```

Clone the repo if needed:

```bash
git clone <YOUR_REPO_URL> chefmate-iac
cd /work/chefmate-iac
```

## 2. Tooling And Auth Preflight

Confirm the control host can run the required tools.

```bash
export PATH=/work/.local/bin:$PATH
terraform version
ansible-playbook --version | head -1
```

If Terraform is missing in Jupyter, install it as in the lab materials. If Ansible is missing, install it there too.

Prepare Terraform auth exactly as the Chameleon Jupyter environment requires:

```bash
cd /work/chefmate-iac/tf/kvm
export PATH=/work/.local/bin:$PATH
unset $(set | grep -o "^OS_[A-Za-z0-9_]*")
```

Place `clouds.yaml` in this directory:

```bash
cp /path/to/clouds.yaml /work/chefmate-iac/tf/kvm/clouds.yaml
```

To make the OpenStack CLI reliably use the same application-credential auth as Terraform, also set:

```bash
export OS_CLIENT_CONFIG_FILE=/work/chefmate-iac/tf/kvm/clouds.yaml
export OS_CLOUD=openstack
```

Quick auth check:

```bash
openstack --os-cloud openstack token issue
```

If you see an error like `Missing value auth-url required for auth plugin password`, the CLI is not reading `clouds.yaml` correctly. Re-run the `unset` command above, set `OS_CLIENT_CONFIG_FILE`, and use `--os-cloud openstack` explicitly on `openstack` commands.

## 3. Choose The Network Strategy

This is where the project diverges from blindly copying the lab.

### Preferred Pattern

The project only needs one public entrypoint. Use this pattern:

1. all nodes are on one project-controlled cluster network
2. `node1` gets the single floating IP
3. `node1` is the SSH jump host and user entrypoint
4. services are exposed through ingress, controlled node ports, or selective port-forwarding

### If `sharednet1` Works

You may use the existing Terraform flow if `sharednet1` has capacity.

### If `sharednet1` Is Exhausted

Do not keep trying to force the lab assumption. Move to a project-owned routed network:

1. create a private project network and subnet
2. create a router with external gateway `public`
3. attach the subnet to the router
4. attach all nodes to that network
5. assign one floating IP to `node1`

This still satisfies the project rubric and Chameleon guidance.

## 4. Terraform Bring-Up

Go to the Terraform directory:

```bash
cd /work/chefmate-iac/tf/kvm
```

Create your working vars file:

```bash
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars` and set:

1. `suffix`
2. `key`
3. `security_group_names`
4. `flavor_id` if you created a Chameleon lease and reserved capacity
5. any network-specific values needed by your chosen topology

Then run:

```bash
terraform init
terraform validate
terraform plan
terraform apply -auto-approve
```

If you are using a reservation-backed `flavor_id`, it is safer to apply with one instance at a time:

```bash
terraform apply -auto-approve -parallelism=1
```

Save these results for later use:

1. the floating IP for `node1`
2. private IPs for `node1`, `node2`, `node3`
3. screenshots from Horizon if helpful for evidence

## 5. If Terraform Fails During Infrastructure Bring-Up

Two practical failure modes showed up during real deployment.

### `sharednet1` fixed-IP exhaustion

The symptom is:

- `IpAddressGenerationFailure`
- no more IP addresses available on `sharednet1`

Use this quick diagnostic from the Jupyter terminal:

```bash
openstack --os-cloud openstack port create \
  --network sharednet1 \
  --security-group default \
  sharednet1-smoke-<projid>
```

Interpretation:

1. if this fails with the same error, `sharednet1` is the problem
2. if it succeeds, delete the smoke-test port and retry Terraform

Delete it with:

```bash
openstack --os-cloud openstack port delete sharednet1-smoke-<projid>
```

If the smoke test confirms `sharednet1` exhaustion, switch to the project-owned routed network pattern described above.

### Compute placement failure

The symptom is a server fault like:

- `No valid host was found. There are not enough hosts available.`

This is a compute-capacity issue, not a networking issue. The practical fix is:

1. create a short Chameleon lease from the Jupyter terminal
2. get the reserved flavor UUID from the lease
3. set `flavor_id` in `terraform.tfvars`
4. retry with `terraform apply -auto-approve -parallelism=1`

Example lease flow:

```bash
openstack --os-cloud openstack reservation lease create lease_mlops_<projid> \
  --start-date "$(date -u -d '+10 seconds' '+%Y-%m-%d %H:%M')" \
  --end-date "$(date -u -d '+12 hours' '+%Y-%m-%d %H:%M')" \
  --reservation "resource_type=flavor:instance,flavor_id=$(openstack --os-cloud openstack flavor show m1.large -f value -c id),amount=3"

flavor_id=$(openstack --os-cloud openstack reservation lease show lease_mlops_<projid> -f json -c reservations | jq -r '.reservations[0].flavor_id')
echo "$flavor_id"
```

Then put that UUID in `terraform.tfvars` as `flavor_id = "..."`.

## 6. Update Ansible Config

Go to the Ansible directory:

```bash
cd /work/chefmate-iac/ansible
cp ansible.cfg.example ansible.cfg
```

Edit `ansible.cfg`:

1. replace `REPLACE_WITH_FLOATING_IP` with the actual Terraform floating IP for `node1`
2. keep the repo path aligned with where the repo lives on the control host

If your Jupyter environment uses a newer `ansible-core`, make sure the config uses:

```ini
[defaults]
stdout_callback = default
callback_result_format = yaml
```

The older `stdout_callback = yaml` setting can fail with an error about `community.general.yaml` having been removed.

## 7. Confirm Inventory

The default inventory assumes:

- `node1 -> 192.168.1.11`
- `node2 -> 192.168.1.12`
- `node3 -> 192.168.1.13`

If Terraform created different private IPs, update both files:

1. `ansible/inventory.yml`
2. `ansible/k8s/inventory/mycluster/hosts.yaml`

## 8. Verify Connectivity

From the Jupyter terminal:

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml general/hello_host.yml
```

Expected result:

1. all three nodes respond
2. hostnames print correctly

If the first run fails right after VM creation, warm up the jump path manually and retry:

```bash
ssh -J cc@<FLOATING_IP> -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null cc@192.168.1.11 hostname
ssh -J cc@<FLOATING_IP> -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null cc@192.168.1.12 hostname
ssh -J cc@<FLOATING_IP> -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null cc@192.168.1.13 hostname
```

If those SSH checks work, Ansible should work from Jupyter as the control host.

## 9. Run Pre-Kubernetes Setup

Before running this step, make sure every node has outbound package access. The playbook installs packages with `apt`, so workers cannot be private-only hosts with no route to the Internet.

Two valid patterns are:

1. all nodes have a `sharednet1` interface for outbound NAT, with only `node1` holding the floating IP
2. all nodes sit on a project-owned routed network that provides outbound access

If only `node1` has `sharednet1` and the private subnet is created with `no_gateway = true`, `node2` and `node3` will typically fail during `apt update` in this step.

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml pre_k8s/pre_k8s_configure.yml
```

Expected result:

1. base packages installed
2. system settings applied
3. the nodes are ready for Kubernetes bootstrap

## 10. Bootstrap Kubernetes

```bash
cd /work/chefmate-iac/ansible/k8s/kubespray
ansible-playbook -i ../inventory/mycluster/hosts.yaml --become --become-user=root ./cluster.yml
```

Expected result:

1. control-plane services come up on `node1`
2. `node2` and `node3` join the cluster
3. the final recap shows success

## 11. Run Post-Kubernetes Setup

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml post_k8s/post_k8s_configure.yml
```

Expected result:

1. `kubectl` is configured for the `cc` user
2. Helm is installed
3. cluster verification commands succeed

## 12. Deploy Platform Services

Deploy `Postgres + MinIO + MLflow`:

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml argocd/argocd_add_platform.yml
```

Expected result:

1. namespace exists
2. Kubernetes secrets are created without storing credentials in Git
3. the platform release installs successfully

## 13. Deploy Mealie

Deploy the open-source service used for `Q2.3`:

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml argocd/argocd_add_staging.yml
```

Expected result:

1. Mealie namespace exists
2. Mealie credentials secret is created
3. the release installs successfully

## 14. Verify The Running Cluster

SSH to `node1` if needed:

```bash
ssh cc@<FLOATING_IP>
```

Then run:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get pvc -A
kubectl top nodes
kubectl top pods -A
```

Check specifically that:

1. the cluster nodes are `Ready`
2. platform pods are healthy
3. Mealie pods are healthy
4. PVCs are `Bound`
5. service exposure matches your selected network strategy

## 15. Browser And Service Verification

Use the floating IP of `node1` plus the ports or ingress rules you actually configured.

Examples if using the current port-based layout:

- Mealie: `http://<FLOATING_IP>:8082`
- MLflow: `http://<FLOATING_IP>:8000`
- MinIO Console: `http://<FLOATING_IP>:9001`

If direct browser access to the floating IP is flaky but the service is working, you can validate from a local browser through an SSH tunnel to the real Chameleon deployment:

```bash
ssh -L 8082:10.56.2.160:8082 cc@<FLOATING_IP>
ssh -L 8000:10.56.2.160:8000 cc@<FLOATING_IP>
ssh -L 9001:10.56.2.160:9001 cc@<FLOATING_IP>
```

Then open:

- Mealie: `http://localhost:8082`
- MLflow: `http://localhost:8000`
- MinIO Console: `http://localhost:9001`

If a service does not load:

1. check `kubectl get svc -A`
2. check `kubectl get pods -A`
3. check service type, node port, or ingress rule
4. check security group rules on the public entrypoint

## 16. Persistence Check

You must show durability for `Q2.4`.

Record:

```bash
kubectl get pvc -A
```

Optional stronger evidence:

```bash
kubectl delete pod -n chefmate-platform -l app=mlflow
kubectl get pods -n chefmate-platform -w
kubectl get pvc -A
```

The goal is to show that state is backed by persistent storage, not by an ephemeral container filesystem.

## 17. Fill The Q2.1 Evidence Table

Use this file as the source:

- `/work/chefmate-iac/ansible/post_k8s/RIGHT_SIZING.md`

Make sure the table reflects the resource values actually configured in the manifests:

- `/work/chefmate-iac/k8s/platform/values.yaml`
- `/work/chefmate-iac/k8s/staging/values.yaml`

Populate it with real observations from:

```bash
kubectl top nodes
kubectl top pods -A
kubectl get pods -A -o wide
kubectl get pvc -A
```

The final PDF should cover the services you actually deployed, for example:

1. `mealie`
2. `mealie-postgres`
3. `mlflow`
4. `minio`
5. `postgres`

For the rubric, the important link is:

1. requests and limits come from the chart values you set in Git
2. right-sizing justification comes from the live Chameleon measurements and healthy runtime behavior

## 18. Record Q2.3 Video

Show this sequence:

1. `kubectl get nodes`
2. `kubectl get pods -A`
3. `kubectl get svc -A`
4. browser open to Mealie
5. one basic Mealie interaction

The goal is to prove the selected open-source service runs in Kubernetes on Chameleon and is reachable.

## 19. Record Q2.4 Video

Show this sequence:

1. `kubectl get pods -A`
2. `kubectl get pvc -A`
3. browser open to MLflow
4. browser open to MinIO console
5. optional pod restart and persistence proof

The goal is to prove shared platform services run in Kubernetes on Chameleon with persistent storage.

## 20. Create The Q2.2 Archive

From the parent directory:

```bash
cd /work
tar -czf chefmate-q2-infra.tgz chefmate-iac
```

Or:

```bash
cd /work
zip -r chefmate-q2-infra.zip chefmate-iac
```

If you prefer not to submit markdown notes, create the archive with your preferred exclusions before upload.

Before creating the archive, confirm no live secret-bearing files are included, especially:

1. `tf/kvm/clouds.yaml`
2. `tf/kvm/terraform.tfvars`
3. `ansible/ansible.cfg`

## 21. Suggested Q2.2 Text Box Content

Use this sequence in the submission text field:

1. Provision infrastructure with Terraform from `tf/kvm/`
2. Configure SSH jump-host access in `ansible/ansible.cfg` using the floating IP of `node1`
3. Verify connectivity with `general/hello_host.yml`
4. Prepare nodes with `pre_k8s/pre_k8s_configure.yml`
5. Bootstrap Kubernetes with `ansible/k8s/kubespray/cluster.yml`
6. Run `post_k8s/post_k8s_configure.yml` for cluster tooling
7. Deploy shared platform services with `argocd/argocd_add_platform.yml`
8. Deploy Mealie with `argocd/argocd_add_staging.yml`
9. Validate health with `kubectl get pods -A`, `kubectl get svc -A`, browser checks, and PVC evidence

## 22. Final Submission Checklist

### Q2.1
- PDF exported from `ansible/post_k8s/RIGHT_SIZING.md`
- real Chameleon evidence included

### Q2.2
- `.zip` or `.tgz` of the infrastructure repo
- text field filled with bring-up steps

### Q2.3
- video of Mealie running in Kubernetes on Chameleon

### Q2.4
- video of MLflow and MinIO running on Chameleon
- persistence shown via PVCs or equivalent durable backing storage

## 23. Troubleshooting

### Terraform auth fails
Check:

1. `clouds.yaml` is present in `tf/kvm/`
2. `export PATH=/work/.local/bin:$PATH` has been run
3. `unset $(set | grep -o "^OS_[A-Za-z0-9_]*")` has been run
4. `export OS_CLIENT_CONFIG_FILE=/work/chefmate-iac/tf/kvm/clouds.yaml` has been run
5. you are using `--os-cloud openstack` on `openstack` CLI commands

If you see `Missing value auth-url required for auth plugin password`, the CLI has fallen back to the wrong auth mode.

### Terraform fails on public networking
Check:

1. whether `sharednet1` is exhausted
2. whether you still have stale ports or floating IPs from earlier attempts
3. whether you should move to a project-owned routed network instead of forcing the lab pattern

### Terraform fails on instance creation
Check:

1. the server `fault` from `openstack --os-cloud openstack server show <server-id> -f yaml`
2. whether the fault says `No valid host was found`
3. whether you need a reservation-backed `flavor_id`
4. whether you should retry with `terraform apply -auto-approve -parallelism=1`

### Ansible cannot connect
Check:

1. `ansible.cfg`
2. floating IP
3. SSH keypair
4. private IP inventory
5. whether a manual `ssh -J cc@<FLOATING_IP> cc@192.168.1.1x hostname` works for each node

If Ansible errors on `community.general.yaml`, update `ansible.cfg` to use `stdout_callback = default` and `callback_result_format = yaml`.

### Pre-Kubernetes package install fails
Check:

1. whether `node2` and `node3` have outbound Internet/NAT access
2. whether only `node1` has `sharednet1` while the private subnet has `no_gateway = true`
3. whether workers can `apt update` from the shell

The most direct fix is to give all nodes outbound networking while keeping only one floating IP on `node1`.

### Kubernetes bootstrap fails
Check:

1. node-to-node private connectivity
2. control-plane readiness on `node1`
3. worker join status

### Service deployment fails
Check:

1. namespace existence
2. secret creation
3. `kubectl describe pod ...`
4. `kubectl logs ...`
5. service exposure and firewall or security-group rules

### Mealie works locally but not through the floating IP
Treat this as an exposure-path issue first, not as proof that Mealie is broken.

Check:

1. `kubectl -n chefmate-staging get pods -o wide`
2. `kubectl -n chefmate-staging logs deploy/mealie --tail=100`
3. `kubectl -n chefmate-staging get svc mealie -o wide`
4. whether the security group on `node1` allows TCP `8082`
5. whether the app is already reachable through an SSH tunnel or browser session on the control host

If logs still show a timeout from Mealie to `mealie-postgres`, the likely cause is pod placement plus `local-path` PVC locality. In that case, the clean fix is:

```bash
kubectl delete namespace chefmate-staging
```

Wait for the namespace to disappear, then redeploy from the control host:

```bash
cd /work/chefmate-iac/ansible
ansible-playbook -i inventory.yml argocd/argocd_add_staging.yml
```

Re-check:

```bash
kubectl -n chefmate-staging get pods -o wide
kubectl -n chefmate-staging logs deploy/mealie --tail=100
```

## 24. Minimum Success Condition

You are done when all of these are true:

1. `kubectl get pods -A` shows healthy platform and app pods
2. `kubectl get pvc -A` shows bound PVCs or equivalent durable storage
3. Mealie is reachable and functional from the browser
4. MLflow is reachable and functional from the browser
5. MinIO is reachable and functional from the browser
6. `RIGHT_SIZING.md` is filled with real measurements and exported to PDF
7. the repo archive is created
8. both videos are recorded
