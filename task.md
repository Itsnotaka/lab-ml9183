# Q2 DevOps / Platform (Gradescope)

**Total: 9 points**

## Requirements (credit)

To receive credit for the DevOps/Platform deliverables:

- **Kubernetes** is required for **4-person** teams. (For **3-person** teams, Kubernetes is optional; Docker Compose is acceptable.)
- **Git as source of truth:** IaC/CaC artifacts and Kubernetes manifests (or equivalents) are in the repo.
- **Durability:** Platform state and artifacts persist across pod restarts (MLflow artifacts and other shared artifacts use persistent volume/object storage, not ephemeral container filesystems).
- **Secrets hygiene:** No secrets in Git.

---

## Q2.1 — Infrastructure requirements table

**Points: 1**

**Prompt:** For each service running in your cluster, show the GPU, CPU, memory **requests** and **limits** you set, plus brief evidence from Chameleon showing how you arrived at appropriate values for right-sizing.

**Submit:** PDF document.

---

## Q2.2 — Repository artifacts

**Points: 4**

**Submit:**

- IaC/CaC materials that provision Chameleon infrastructure and configure a Kubernetes cluster (cluster, networking/ingress, persistent volumes, namespaces).
- Kubernetes manifests and other necessary materials to deploy the **open source service** the project is based on, and to deploy **platform services required by other team members**.
- A `.zip` or `.tgz` of your infrastructure repository (or repositories, if more than one).

**Text field:** Explain how to use the submission (sequence of steps to bring up the system).

> **Note:** There is no credit for systems that are **not** running on Chameleon Cloud. Code or configuration that has not been executed in the target environment, or that only runs locally, does not count. Submit only items you have implemented and executed on Chameleon.

---

## Q2.3 — Demo video: open source service

**Points: 2**

**Submit:** Sped-up demo video of the selected open source service **running inside Kubernetes on Chameleon**.

The video should demonstrate everything from launching the service to confirming its health status in Kubernetes to validating in a browser that it is reachable and functional. **No longer than a few minutes.**

> **Note:** Same as Q2.2 — must be running on Chameleon; local-only work does not count.

---

## Q2.4 — Demo video: shared platform services

**Points: 2**

**Submit:** Sped-up demo video of **shared platform services** running inside Kubernetes on Chameleon, **with persistent storage** as appropriate.

Same demonstration expectations as Q2.3 (launch → health in K8s → browser/functional check). **No longer than a few minutes.**

> **Note:** Same Chameleon execution requirement as Q2.2–Q2.3.

---

## Q3 — Bonus (optional)

**Points: 0**

If attempting extra credit:

- Integrate a **platform tool/framework not used in the lab assignments** that materially improves operability, with a **realistic example** and justification (e.g. secrets management, TLS automation, centralized logging, image security/scanning, distributed tracing). **Prometheus/Grafana do not count** (Lab 8).
- Show **one concrete operational win** in the demo video plus a short justification.
- Submit the demo video and **PDF** explaining the example, plus any artifacts (e.g. Dockerfiles, Python scripts).
