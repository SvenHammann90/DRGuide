# Chapter 5 — Kubernetes from zero

You keep hearing about Kubernetes because it's how most modern companies run their software, and because it has its own security model that sits between the cloud and the application. This chapter explains it from scratch. No prior knowledge assumed beyond "a container is a packaged application" (Chapter 4).

## The problem Kubernetes solves

Suppose you have a hundred services, each packaged as a container, and each needing several copies running for reliability. You have a few hundred machines to run them on. Questions immediately arise:

- Which container should run on which machine?
- What happens when a machine dies? Who restarts its containers elsewhere?
- How do containers find and talk to each other when they keep moving between machines?
- How do you roll out a new version without downtime?
- How do you give each service its configuration and secrets?

Doing this by hand is impossible at scale. **Kubernetes** (often written **K8s**, for the eight letters between K and s) is a system that does it for you. You tell it what you want running, and it figures out where and how, and keeps it that way.

A good one-line description: **Kubernetes is an operating system for a fleet of machines.** A normal operating system decides which programs run on which CPU cores. Kubernetes decides which containers run on which machines.

## The core idea: desired state

The most important concept in Kubernetes is **declarative desired state**.

You don't tell Kubernetes "start three copies of the web server on machines 4, 7, and 12." You tell it "I want three copies of the web server running." Kubernetes writes that down, then constantly compares what you asked for with what's actually running, and fixes any difference. If a machine dies and one copy disappears, Kubernetes notices that there are two instead of three and starts another one somewhere else.

This loop (compare desired to actual, then act to close the gap) is called **reconciliation**, and the components that do it are called **controllers**. Almost everything in Kubernetes works this way.

For security, this has an interesting consequence: **if an attacker changes the desired state, Kubernetes will faithfully maintain their change.** Add a malicious container to the desired state and Kubernetes will keep it running, restart it if it crashes, and move it if its machine dies. Persistence for free.

## The parts of a cluster

A Kubernetes installation is called a **cluster**. It has two halves.

### The control plane: the brain

- **API server:** the front door. Every action in Kubernetes, by people, by tools, and by Kubernetes' own components, is a request to the API server. This is the most important component for security: if you control the API server's answers, you control the cluster. It's also what produces the **audit log**.
- **etcd:** the database where the cluster stores its desired state and all its configuration, including secrets.
- **Scheduler:** decides which machine each new container should run on.
- **Controller manager:** runs the reconciliation loops.

In managed Kubernetes (AWS EKS, Google GKE, Azure AKS), the cloud provider runs the control plane for you. You never see those machines, but you can call the API server, and you can (and should) turn on its audit logs.

### The nodes: the muscle

**Nodes** are the machines (usually cloud VMs) that actually run containers. Each node runs:

- **kubelet:** an agent that talks to the API server, receives instructions ("run these containers"), and makes them happen on its machine.
- **A container runtime:** the software that actually starts containers.

Groups of similar nodes are often called **node pools** (for example, a pool of normal machines for web services and a pool of GPU machines for model inference).

Notice the shape: the API server is the center of everything, and the kubelets on each node take instructions from it. An attacker who can talk to the API server with enough permissions can run anything on any node. That's why the API server's access control matters so much.

## The objects you'll hear about

Everything in Kubernetes is described as an **object** in the API, usually written as a YAML configuration file. The main ones:

**Pod.** The smallest unit Kubernetes runs. A pod is one or more containers that run together on the same node and share a network address. In practice, "a pod" usually means "one running copy of a service." Pods are disposable: they get created and destroyed constantly.

**Deployment.** A desired state for a set of identical pods: "run three copies of this web server image, and when I change the image, roll them over gradually." You rarely create pods directly. You create deployments, and the deployment controller creates the pods.

**DaemonSet.** "Run one copy of this pod on every node." Used for things like log collectors and security agents. Also useful to attackers, since a malicious DaemonSet gets code onto every node at once.

**Job and CronJob.** Pods that run to completion (a batch task) or on a schedule. CronJobs are the Kubernetes equivalent of scheduled tasks, and a classic persistence mechanism.

**Service.** A stable network name and address for a set of pods, so other pods can reach them even though the pods themselves come and go.

**Ingress.** Rules for routing traffic from outside the cluster to services inside it. This is often where the cluster meets the internet.

**Namespace.** A way to divide one cluster into separate areas, typically per team or per application. Namespaces scope names and permissions. They are **not** a strong security boundary by themselves: by default, pods in different namespaces can talk to each other over the network, and they may share nodes.

**ConfigMap and Secret.** Ways to give configuration to pods. A Secret is meant for sensitive values like passwords and keys. Important detail: by default, Kubernetes Secrets are only **base64-encoded**, which is an encoding, not encryption. Anyone with permission to read Secrets through the API can read their values. Encryption of Secrets at rest in etcd is a separate setting (managed providers often enable it, or offer it), and many teams store real secrets in an external secret manager instead.

## Identity in Kubernetes

Kubernetes has its own access control, separate from (but connected to) cloud IAM.

**Who talks to the API server:**

- **Humans:** engineers using the `kubectl` command-line tool. In managed clusters, they usually authenticate with their cloud identity (which came from SSO), and the cluster maps that to Kubernetes permissions.
- **Workloads:** pods that need to call the Kubernetes API (for example, a controller that manages other pods). They use **service accounts**, which are Kubernetes identities. Note the name collision: a Kubernetes service account is not the same as a GCP service account, though they're often linked.
- **Kubernetes' own components:** kubelets, controllers, and so on, each with their own identity.

**How pods get their identity:** By default, Kubernetes puts a **token** for the pod's service account inside the pod as a file (at `/var/run/secrets/kubernetes.io/serviceaccount/token`). Any process in the pod can read it and use it to call the API server. Modern versions make these tokens short-lived and tied to the specific pod, which helps. But if a pod is compromised, the attacker gets whatever permissions its service account has. Pods that don't need the API can, and should, be configured not to receive the token at all.

**Authorization: RBAC.** Kubernetes uses **role-based access control (RBAC)**:

- A **Role** lists allowed actions on resources within one namespace ("may read pods and logs in namespace `web`").
- A **ClusterRole** does the same across the whole cluster.
- A **RoleBinding** or **ClusterRoleBinding** grants a Role or ClusterRole to a user, group, or service account.

The most powerful ClusterRole is `cluster-admin`, which can do anything. Granting it is roughly equivalent to giving someone root on every node.

**Connecting to the cloud: workload identity.** Pods often need cloud permissions too (to read a bucket, for example). The modern way is **workload identity** (GKE Workload Identity, EKS Pod Identity or IRSA, Azure Workload Identity): a Kubernetes service account is linked to a cloud identity, and pods using that service account can exchange their Kubernetes token for cloud credentials. This avoids putting cloud keys in Secrets, but it creates a direct bridge: **control of a pod with a linked service account means control of that cloud identity.**

The older approach, where pods simply use the node's cloud identity through the metadata service (Chapter 4), is worse: every pod on the node shares the node's permissions, which are often broad.

## Admission control: the gatekeeper

Before the API server accepts a new object (like a pod), it can pass the request through **admission controllers**. These can reject or modify requests based on policy:

- "Reject pods that ask to run as privileged."
- "Reject pods using images from registries outside our organization."
- "Reject pods that mount the host's filesystem."

Kubernetes has a built-in system called **Pod Security Standards** with three levels (privileged, baseline, restricted). Many teams add policy engines like **OPA Gatekeeper** or **Kyverno** for custom rules. Admission control is the main prevention layer for Kubernetes, and Chapter 6 shows why it matters.

## How changes reach a cluster

In a mature setup, people rarely change clusters by hand. Changes flow through a pipeline:

- Engineers change YAML files in a code repository.
- A deployment system (often a **GitOps** tool like Argo CD or Flux) watches the repository and applies changes to the cluster automatically.

This is good for D&R: the repository becomes a record of intended state, and anything in the cluster that *doesn't* match the repository is suspicious. It also means the deployment system has powerful cluster permissions, which makes it a high-value target (Chapter 9).

## Putting it together

A request's journey, to tie the parts together:

1. An engineer changes a deployment file to use a new image version and merges it.
2. The GitOps tool sees the change and sends an update request to the API server, using its service account.
3. The API server authenticates the tool, checks RBAC (is this service account allowed to update deployments here?), runs admission control (is this image from an allowed registry? does the pod spec follow policy?), records the request in the audit log, and stores the new desired state in etcd.
4. The deployment controller notices the desired state changed and creates new pods.
5. The scheduler assigns each new pod to a node.
6. The kubelet on each node starts the containers.
7. The pods start up, read their service account token, exchange it for cloud credentials through workload identity, and begin serving traffic.

Each step is a place where an attacker might intervene, and a place where you might see them. That's the subject of the next chapter.

## Questions to think about

1. Why does Kubernetes' "desired state" model make persistence easy for an attacker who gets API access? What would you look for to find it?
2. In your own words, what's the difference between a Kubernetes service account, a cloud service account, and a human engineer's access to a cluster?
3. Namespaces are not a strong security boundary. What would you need in addition to make two teams' workloads in the same cluster truly isolated from each other?
