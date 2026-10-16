# Chapter 6 — Attacking and defending Kubernetes

With the parts from Chapter 5 in mind, this chapter follows an attacker through a cluster: how they get in, how they move, what they're after, what you'd see, and what would have stopped them.

## How attackers get into a cluster

Common entry points, roughly from most to least frequent:

1. **A vulnerable application running in a pod.** A web service with a remote code execution bug, an SSRF bug, or a deserialization flaw. The attacker gets code running inside one container. This is the most common starting point.
2. **Stolen cluster credentials.** A `kubeconfig` file from a developer laptop, a CI system's cluster credentials, or a cloud identity that's mapped to cluster permissions.
3. **An exposed control plane or component.** An API server reachable from the internet with weak authentication, an exposed dashboard, or an exposed kubelet API. Managed clusters have made this rarer, but misconfigurations still happen.
4. **A malicious image.** A compromised base image or dependency, so the attacker's code arrives inside a legitimate deployment.
5. **A compromised deployment pipeline.** If the attacker controls what gets deployed, they don't need to break in at all (Chapter 9).

## The classic attack path

Here's the path that most Kubernetes attack discussions build on. Starting from code execution in one pod:

**Step 1: Look around inside the pod.** The attacker checks what they have: environment variables (often containing secrets), mounted files, and the service account token at the well-known path. They check whether they're running as root, whether the container is privileged, and what's mounted from the host.

**Step 2: Use the service account token.** They call the API server with the pod's token and ask "what am I allowed to do?" (Kubernetes even has an API for this question.) If the service account has broad permissions, such as listing Secrets cluster-wide or creating pods, the attacker has a path forward.

**Step 3: Read secrets.** If they can list Secrets, they read every password, API key, and credential stored in the cluster. This often yields database passwords, cloud credentials, and tokens for other service accounts, some more privileged than the one they started with.

**Step 4: Create a pod that escapes to the node.** If they can create pods, they create one designed to break isolation: privileged, with the node's filesystem mounted, sharing the node's process space and network. From inside that pod, they effectively have root on the node.

**Step 5: Take over the node.** On the node, they can see and enter every other container running there, read their service account tokens and secrets, and access the node's own credentials: the kubelet's credentials and the node's cloud identity from the metadata service.

**Step 6: Jump to the cloud.** Using the node's cloud identity, or a workload identity token taken from another pod, they call the cloud API. Now they're out of Kubernetes and into the wider cloud environment, where Chapters 2 and 3 apply.

**Step 7: Persist.** They create a DaemonSet or CronJob that keeps their code running, add a ClusterRoleBinding granting themselves admin, or plant a modified image. Thanks to reconciliation, Kubernetes will keep their changes running.

Not every intrusion goes through every step, and many stop early if permissions are tight. But each step is a checkpoint where you can detect or prevent.

## What you'd see

Kubernetes gives you three layers of telemetry, and an attacker usually touches all three.

### The Kubernetes audit log

Every request to the API server can be recorded: who made it, from which address, which action, on which object, and whether it was allowed. In managed clusters it has to be turned on and shipped somewhere (it usually lands in the cloud provider's logging service).

High-value things to detect:

- **`exec` into a pod** in production. Engineers occasionally do this for debugging, which is exactly why it should be rare, logged, and reviewed. An `exec` by a service account, rather than a human, is especially suspicious.
- **Creating a pod with dangerous settings:** privileged mode, host filesystem mounts, host network or process namespaces. (Admission control should block most of these. A detection catches the exceptions and any bypass.)
- **New ClusterRoleBindings,** especially to `cluster-admin`, and especially outside the normal deployment pipeline.
- **A service account listing or reading Secrets** that it doesn't normally touch, or listing Secrets across all namespaces.
- **Permission probing:** a burst of denied requests from one service account (the Kubernetes version of the access-denied spike from Chapter 3).
- **Anonymous or unauthenticated requests** that succeed.
- **New DaemonSets or CronJobs** created outside the pipeline.
- **Changes made directly to the cluster** instead of through GitOps, when your environment normally uses GitOps.

The common thread: in a well-run cluster, **most changes come from the deployment pipeline's identity.** Changes from any other identity are worth a look.

### Runtime telemetry inside containers and nodes

The audit log only sees API calls. It doesn't see what happens inside a container. For that you need runtime security tools, which watch process execution, file access, and network connections on each node. Open-source **Falco** is a well-known example, and most commercial cloud security tools include something similar.

High-value things to detect:

- **A shell or unexpected process started inside a container,** especially one whose image normally runs a single program.
- **Reading the service account token file** by a process that isn't the application.
- **Downloading and running a binary** inside a running container (containers shouldn't change after they start).
- **Connections to the metadata service** from pods that shouldn't need it.
- **Unexpected outbound connections,** for example to the internet from a pod that normally only talks internally, or to known mining pools.
- **Signs of container escape:** processes in a container accessing host paths, or loading kernel modules.

### Cloud logs

When the attacker jumps from the cluster to the cloud, the cloud audit log takes over. The key signals:

- **The node's cloud identity making unusual calls** (nodes normally make a narrow, predictable set of calls).
- **Workload identity credentials used from outside the cluster,** the Kubernetes version of "instance credentials used from an external IP."
- **A pod's linked cloud identity accessing data it normally doesn't.**

Connecting these three layers (which pod, on which node, using which service account, linked to which cloud identity, made which cloud call) is the hard part of Kubernetes investigations, and the place where good data engineering pays off.

## What prevents it

Most of the attack path above is preventable with configuration. For each step, the corresponding control:

| Attacker step | Prevention |
|---|---|
| Read the service account token | Don't mount tokens into pods that don't need the API. |
| Use the token for broad API access | Least-privilege RBAC. Most application pods need no API permissions at all. |
| Read all Secrets | Restrict who can read Secrets. Encrypt Secrets at rest. Prefer an external secret manager. |
| Create a privileged pod | Admission control: enforce Pod Security Standards (baseline or restricted) and block privileged pods, host mounts, and host namespaces. |
| Escape to the node | Run containers as non-root, with minimal Linux capabilities and read-only filesystems. Use sandboxed runtimes (like gVisor) for untrusted workloads. |
| Use the node's cloud identity | Block pods from reaching the metadata service. Give nodes minimal cloud permissions. Use workload identity instead of node identity. |
| Move between pods | **Network policies:** rules for which pods may talk to which. By default, everything can talk to everything. |
| Run attacker images | Only allow images from your own registries, ideally signed and verified. |
| Persist via the API | Changes only through the pipeline (GitOps), with alerts on direct changes. Short-lived human access to clusters. |

Note how many of these are about **removing defaults.** Kubernetes defaults favor convenience: tokens mounted everywhere, all pods able to talk to each other, any image allowed. A secure cluster is mostly a cluster where someone deliberately turned those defaults off.

## The multi-tenant problem

A question that comes up in AI labs and other compute-heavy companies: can different workloads, with different trust levels, safely share a cluster?

Namespaces plus RBAC plus network policies give a reasonable separation for workloads that roughly trust each other (different teams at the same company). They're **not** a strong boundary against a determined attacker, because workloads still share nodes and a kernel. For workloads that must be strongly separated (for example, a system holding model weights vs a system running untrusted code), the safer options are separate node pools at minimum, sandboxed runtimes, or entirely separate clusters, ideally in separate cloud accounts (Chapter 1).

As a D&R engineer, it's worth knowing which workloads share infrastructure with which. "What else runs on the same nodes as the thing that got compromised?" is a key scoping question.

## Questions to think about

1. An attacker has code execution in a pod. Its service account can list pods but nothing else. Admission control blocks privileged pods. What can they still do, and what would you look for?
2. Why is "a change made directly to the cluster, not through GitOps" such a valuable detection? When would it produce false positives?
3. Your runtime tool alerts on a shell inside a production pod. The audit log shows an engineer ran `exec` into that pod two minutes earlier. Is the case closed? What else would you check?
