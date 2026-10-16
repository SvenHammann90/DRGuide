# Chapter 0 — The cloud mental model

If you've never operated cloud infrastructure, the vocabulary can feel like a wall: VPCs, IAM roles, node pools, service accounts, buckets, KMS, OIDC federation. Underneath it, though, there's one simple idea that makes the rest fall into place. This chapter is about that idea, and about the map of layers the rest of the guide follows.

## Everything is an API call

In a traditional data center, infrastructure is physical. To get a new server, someone racks it. To change the network, someone reconfigures a switch. To access a machine, you're on the network it's on.

In the cloud, **all infrastructure is created, changed, and destroyed through API calls**. A virtual machine is the result of a "create instance" request. A storage bucket is a "create bucket" request. Granting a person access is a "set policy" request. The cloud console website, the command-line tools, and infrastructure-as-code tools like Terraform are all just different ways of sending those same API requests.

Three consequences follow, and they shape all of cloud security:

**1. Whoever can call the API controls the infrastructure.** There's no physical barrier. If an attacker holds a credential that the cloud API accepts, they can do whatever that credential is allowed to do, from anywhere on the internet. That's why the first guide said "identity is the perimeter." In the cloud this is literally true.

**2. Almost everything can be logged.** Because every action is an API request, the provider can record it: who called, from where, which action, on which resource, whether it succeeded. These **audit logs** are the most important telemetry in cloud D&R (Chapter 10). Compared to the classic corporate network, where an attacker's movements are reconstructed from fragments, cloud gives you a far more complete record, *if you enabled the right logs*.

**3. Infrastructure is fast and disposable.** Resources are created and destroyed in seconds, often by automation. A cluster might replace all its machines every few days. That's good for security (compromised machines don't live long, and you can rebuild from known-good definitions) and bad for investigation (the machine you want to look at may already be gone).

## Control plane and data plane

One distinction you'll hear constantly:

- **Control plane:** the APIs that *manage* resources. Create a bucket, change who can access it, launch a VM, modify a firewall rule.
- **Data plane:** the actual *use* of resources. Read an object from the bucket, send traffic to the VM, query the database.

They matter differently for D&R:

- Control plane actions are relatively rare, high-signal, and usually logged by default. "Someone made this bucket public" is a control plane event.
- Data plane actions are high-volume and often *not* logged by default, because of cost. "Someone downloaded 10,000 objects from this bucket" is a data plane event.

Many attacks are visible in the control plane (persistence, privilege escalation, logging changes). But **data theft is mostly a data plane event**, which is exactly the part that's easy to leave unlogged. Hold onto that. It comes back in Chapters 7 and 10.

## The shared responsibility model

Cloud providers describe security as split:

- **The provider secures the cloud itself:** physical data centers, the hardware, the hypervisor that separates customers, the underlying services.
- **The customer secures what they put in it:** who has access, how resources are configured, the code they run, the data they store.

In practice almost every cloud breach falls on the customer side: a leaked credential, an over-permissive policy, a public bucket, a vulnerable application. As a defender, you rarely worry about someone breaking AWS itself. You worry about your own configuration and your own identities.

The split shifts with the kind of service. For a raw virtual machine, you own the operating system and everything on it. For a managed database, the provider handles the OS and patching, and you own access and configuration. For a fully managed service, you own little more than who can use it and how.

## A map of the layers

The rest of this guide walks through the layers of a typical cloud environment. Here's the map, roughly from the outside in:

1. **Organization and accounts (Chapter 1):** the top-level structure. Which cloud accounts or projects exist, how they're grouped, and what rules apply to all of them.
2. **Identity and access (Chapters 2 and 3):** who and what can call the API, with which permissions, using which credentials.
3. **Compute (Chapters 4, 5, 6):** where code runs: virtual machines, containers, Kubernetes clusters, serverless functions.
4. **Storage and data (Chapter 7):** where data lives: object storage, databases, disk snapshots, and the encryption keys that protect them.
5. **Network (Chapter 8):** how traffic flows between resources and to the internet.
6. **Delivery pipelines (Chapter 9):** how code and infrastructure changes get into the environment.
7. **Logging and detection (Chapter 10):** what's recorded, and how you watch it.

An attacker's path usually crosses several layers. A typical story: steal a credential (identity) → use it to reach a machine or cluster (compute) → find more credentials there (identity again) → reach the data (storage) → send it out (network). Defending well means knowing what you'd see at each crossing.

## The three providers, briefly

You'll meet three big providers. They solve the same problems with different names:

| Concept | AWS | Google Cloud (GCP) | Microsoft Azure |
|---|---|---|---|
| Unit of isolation | Account | Project | Subscription |
| Grouping | Organization, organizational units | Organization, folders | Tenant, management groups |
| Identity system | IAM (users, roles) | IAM (users, service accounts) | Entra ID (users, managed identities) |
| Audit log | CloudTrail | Cloud Audit Logs | Activity Log, Entra logs |
| Object storage | S3 | Cloud Storage (GCS) | Blob Storage |
| Managed Kubernetes | EKS | GKE | AKS |
| Encryption keys | KMS | Cloud KMS | Key Vault |

This guide mostly uses AWS and GCP examples, because they're the most common in AI infrastructure, but each concept has an equivalent elsewhere. When you hear an unfamiliar service name, the useful question is "which of the layers above is this, and what's its equivalent on the provider I know?"

## Your bridge from account security

You already understand a lot of this from the consumer side:

- A cloud credential is like a session or an API key for a user account, except the "account" can create servers and read every database.
- Cloud IAM is like the permission model of a product's admin roles, but far richer and far more consequential.
- The cloud audit log is like the event stream you'd use for account takeover detection, but for infrastructure.

What's new is mostly the **blast radius**. A compromised consumer account usually hurts one user. A compromised cloud credential can hurt the whole company.

## Questions to think about

1. If every infrastructure change is an API call, what's the cloud equivalent of "an attacker broke into the building"? What has to be true for them to get in?
2. Why is data theft harder to see than privilege escalation in a typical cloud setup? What would you need to change to see it?
3. Think of an API you know from your current work. What would it mean for that API to control your company's servers instead of user accounts? Which of your current detections would you want to keep?
