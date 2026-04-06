Q2 DevOps/Platform contributions
9 Points
Grading comment:
Requirements to get credit for those deliverables:

Kubernetes is required for 4-person teams. (For 3-person teams, Kubernetes is optional; Docker Compose is acceptable.)
Git as source of truth: IaC/CaC artifacts and Kubernetes manifests (or equivalents) are in the repo.
Durability: platform state and artifacts persist across pod restarts (MLflow artifacts and other shared artifacts use a persistent volume/object storage, not ephemeral container filesystems).
Secrets hygiene: no secrets in Git.
Question 2.1 Infrastructure requirements table:
Q2.1 Infrastructure requirements table:
1 Point
Grading comment:
For each service running in your cluster, show the GPU, CPU, memory requests and limits you set, plus brief evidence from Chameleon showing how you arrived at appropriate values for right-sizing.

Upload PDF document.

No file chosenPlease select file(s)Select file(s)
Save Answer
Question 2.1: Infrastructure requirements table:
Question 2.2 Repository artifacts
Q2.2 Repository artifacts
4 Points
Grading comment:
Please upload:

IaC/CaC materials that provision Chameleon infrastructure and configure a Kubernetes cluster (cluster, networking/ingress, persistent volumes, namespaces).
K8S manifests and other necessary materials to deploy the open source service on which the project is based, and to deploy platform services required by other team members.
Create a .zip or .tgz file out of your "infrastructure" repository (or repositories, if you have more than one), and upload it here:

No file chosenPlease select file(s)Select file(s)
Grading comment:
Use this text field to explain how to use it, i.e. what is the sequence of steps to bringing up your system?

Grading comment:
Note: There is no credit for systems that are not running on Chameleon Cloud. Code or configuration that has not been executed in the target environment, or that only runs locally, does not count. You should only submit items that you have implemented and executed on Chameleon.

Save Answer
Question 2.2: Repository artifacts
Question 2.3 Demo video of open source service
Q2.3 Demo video of open source service
2 Points
Grading comment:
Upload a sped-up demo video of the selected open source service running inside Kubernetes on Chameleon.

(Video should demonstrate everything from launching the service to confirming its health status inside K8S to validating in a browser that it is reachable and functional.)

The video should be no longer than a few minutes.

No file chosenPlease select file(s)Select file(s)
Grading comment:
Note: There is no credit for systems that are not running on Chameleon Cloud. Code or configuration that has not been executed in the target environment, or that only runs locally, does not count. You should only submit items that you have implemented and executed on Chameleon.

Save Answer
Question 2.3: Demo video of open source service
Question 2.4 Demo video of shared platform services
Q2.4 Demo video of shared platform services
2 Points
Grading comment:
Upload a sped-up demo video of shared platform services running inside Kubernetes on Chameleon, with persistent storage as appropriate.

(Video should demonstrate everything from launching the service to confirming health status inside K8S to validating in a browser that it is reachable and functional.)

The video should be no longer than a few minutes.

No file chosenPlease select file(s)Select file(s)
Grading comment:
Note: There is no credit for systems that are not running on Chameleon Cloud. Code or configuration that has not been executed in the target environment, or that only runs locally, does not count. You should only submit items that you have implemented and executed on Chameleon.

Save Answer
Question 2.4: Demo video of shared platform services
Q3 Bonus
0 Points
Grading comment:
If you are attempting the extra credit:

Integrate a platform tool/framework not used in the lab assignments that materially improves operability, and justify why it improves your design using a realistic example. You may investigate frameworks for secrets management, TLS automation, centralized logging, image security/scanning, distributed tracing, etc. (Note: Prometheus/Grafana do not "count" because they are used in Lab 8.) You must show one concrete operational win in demo video + a short justification.

Submit the demo video and PDF document explaining, including the concrete realistic example, and also any artifacts (e.g. Dockerfiles, Python scripts) that go along with it.
