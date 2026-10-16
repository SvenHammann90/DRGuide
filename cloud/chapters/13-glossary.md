# Chapter 13 — Glossary

Cloud and Kubernetes terms used in this guide, in plain language. Where providers use different names for the same thing, the equivalents are given.

**Access key (AWS):** a long-lived credential for an IAM user, made of an access key ID (starting `AKIA`) and a secret. Temporary credentials start with `ASIA`.

**Account / project / subscription:** the basic isolation unit in AWS, GCP, and Azure respectively. Resources live inside one.

**Admission controller:** a Kubernetes component that checks or modifies requests before they're accepted, used to enforce policy (for example, "no privileged pods").

**API server:** the central component of a Kubernetes cluster. Every action goes through it, and it produces the audit log.

**Assume a role:** obtain temporary credentials to act as a role, if the role's trust policy allows you.

**Audit log:** the record of API calls. CloudTrail (AWS), Cloud Audit Logs (GCP), Activity Log (Azure), Kubernetes audit log.

**Break-glass access:** emergency credentials for when normal access fails, kept locked away and alerted on when used.

**Bucket:** a container for objects in object storage (S3, GCS, Blob Storage).

**CI/CD:** continuous integration and delivery; the automated pipeline that builds, tests, and deploys code.

**Cluster:** a Kubernetes installation, made of a control plane and nodes.

**ClusterRole / Role:** sets of Kubernetes permissions, cluster-wide or within one namespace. Granted with ClusterRoleBindings and RoleBindings.

**Container:** a packaged application running as an isolated process on a host, sharing the host's kernel.

**Control plane / data plane:** management of resources vs use of resources. In Kubernetes, "control plane" also names the cluster's management components.

**CronJob / DaemonSet / Deployment / Job:** Kubernetes objects that run pods on a schedule, on every node, as a set of replicas, or to completion.

**CSPM / CDR / CWPP / CNAPP:** categories of cloud security products: posture (misconfigurations), detection and response (threats), workload protection (runtime agents), and the bundle of all three.

**Data events / Data Access logs:** audit log records of reading and writing data. Usually off by default.

**Data perimeter:** organization-level rules restricting data access to your identities, your networks, and your resources. Examples: AWS SCPs and resource control policies with organization conditions, GCP VPC Service Controls.

**Drift:** differences between what infrastructure-as-code says should exist and what actually exists.

**Egress:** outbound network traffic.

**Envelope encryption:** encrypting data with a data key that is itself encrypted by a master key held in KMS.

**etcd:** the database where Kubernetes stores its state, including Secrets.

**Federation:** letting identities from one system (like your SSO provider or CI platform) access another (like your cloud) without separate credentials.

**GitOps:** managing deployments by having a tool continuously apply what's in a git repository to the cluster.

**IaC (Infrastructure as Code):** describing infrastructure in files and creating it with tools like Terraform.

**IAM (Identity and Access Management):** the system that decides who can do what in the cloud.

**IMDS (Instance Metadata Service):** the local service on a cloud VM (at `169.254.169.254`) that provides information and role credentials. IMDSv2 is AWS's hardened version.

**Ingress:** in Kubernetes, rules for routing outside traffic into the cluster. In networking generally, inbound traffic.

**JIT (Just-in-time) access:** access granted for a specific task and time window, then automatically removed.

**KMS (Key Management Service):** the service that holds encryption keys and logs their use. AWS KMS, Cloud KMS, Azure Key Vault.

**kubelet:** the agent on each Kubernetes node that runs the pods assigned to it.

**kubeconfig:** the file `kubectl` uses to connect to clusters, often containing credentials.

**Landing zone / account factory:** automation that creates new cloud accounts with a standard secure baseline.

**Least privilege:** granting only the permissions actually needed.

**Managed identity (Azure) / service account (GCP) / role (AWS):** identities for workloads rather than people.

**Management account:** the top account in an AWS organization, which controls all the others.

**Namespace:** a division within a Kubernetes cluster that scopes names and permissions. Not a strong security boundary by itself.

**NAT gateway:** lets private resources make outbound connections without being reachable from the internet.

**Network policy:** a Kubernetes rule for which pods may talk to which.

**Node / node pool:** a machine in a Kubernetes cluster / a group of similar nodes.

**OIDC federation:** exchanging a signed identity token (for example from a CI platform) for cloud credentials, without stored secrets.

**Organization policy / SCP:** guardrails applied across many accounts that override permissions inside them.

**PassRole (AWS):** the permission to attach a role to a resource. A common privilege escalation path.

**Pod:** the smallest unit Kubernetes runs; one or more containers sharing a network address.

**Pod Security Standards:** Kubernetes' built-in policy levels (privileged, baseline, restricted) for how pods may be configured.

**Pre-signed / signed URL:** a temporary link that grants access to a specific object without credentials.

**Principal:** anything that can make a request: a user, role, service account, or workload.

**Privileged container:** a container with nearly all host capabilities, giving little isolation.

**RBAC:** role-based access control, Kubernetes' authorization system.

**Reconciliation:** Kubernetes' continuous process of making actual state match desired state.

**Region:** a geographic location where a cloud provider runs data centers.

**Registry:** where container images are stored and pulled from.

**Resource policy:** a policy attached to a resource (like a bucket) saying who may access it.

**Root user:** the original all-powerful identity of an AWS account.

**Runtime security:** monitoring of process, file, and network activity on hosts and in containers (for example, Falco).

**SBOM:** software bill of materials; a list of components in a piece of software.

**Secret (Kubernetes):** an object for sensitive values. Base64-encoded, not encrypted, by default.

**Secret manager:** a service for storing secrets and handing them to workloads at runtime, with logging (AWS Secrets Manager, GCP Secret Manager, Vault).

**Security group / firewall rule:** rules for what network traffic may reach or leave a resource.

**Serverless:** compute where the provider runs your code on demand without you managing machines (Lambda, Cloud Functions, Cloud Run).

**Service account (Kubernetes):** a Kubernetes identity for pods. Different from a GCP service account, though often linked.

**Service mesh:** infrastructure that manages and secures service-to-service traffic, often requiring authentication for every connection.

**Shared responsibility model:** the provider secures the cloud; the customer secures what's in it.

**Slurm:** a job scheduler common in high-performance and research computing, including some AI training clusters.

**Snapshot:** a point-in-time copy of a disk or database.

**SSRF (Server-Side Request Forgery):** a vulnerability that makes a server fetch a URL chosen by the attacker. Classically used against the metadata service.

**STS (Security Token Service):** the AWS service that issues temporary credentials.

**Subnet (public / private):** a part of a VPC, either reachable from the internet or not.

**Terraform state:** the file in which Terraform records what it manages. Often contains secrets.

**Trust policy:** the part of a role that says who may assume it.

**VPC / VNet:** a private network in the cloud.

**VPC endpoint / Private Service Connect:** private connections from a VPC to cloud services without crossing the internet.

**Workload identity:** linking a Kubernetes service account (or other workload) to a cloud identity so it can get cloud credentials without stored keys.

**Zero trust:** granting access based on verified identity and device, not on network location.

## Further reading

A short list, chosen for building understanding rather than operational detail:

- **The providers' own security documentation:** the AWS Security Reference Architecture, Google Cloud's security foundations blueprint, and Microsoft's cloud security benchmark. Each describes how its provider recommends organizing accounts, identity, logging, and guardrails. Skim the structure, not every detail.
- **Kubernetes documentation, "Security" section:** especially the pages on Pod Security Standards, RBAC good practices, and securing a cluster.
- **The MITRE ATT&CK Cloud and Containers matrices:** to map the behaviors in Chapter 10 to the shared vocabulary.
- **Public cloud breach write-ups and threat reports** from cloud security vendors and the providers themselves, read with the questions from Chapter 11 in mind: initial access, identity chain, what logs showed, what prevention would have changed.
- **RAND, *Securing AI Model Weights* (2024):** for the AI-lab-specific view of protecting weights, now readable with this guide's cloud vocabulary.
