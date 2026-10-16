# Chapter 7 — Storage, data, and encryption

Most attacks end at data. Whether the goal is stealing customer records, research, or model weights, the attacker eventually has to read something and move it somewhere. This chapter covers where data lives in the cloud, how access to it is controlled, how encryption fits in, and the ways data can leave.

## Where data lives

**Object storage** (AWS S3, Google Cloud Storage, Azure Blob Storage). The workhorse. Data is stored as **objects** (files of any size) in **buckets** (named containers). It's cheap, effectively unlimited, and used for everything: application files, backups, logs, datasets, and, at an AI lab, training data and model checkpoints. When people talk about cloud data breaches, they're very often talking about buckets.

**Databases.** Managed relational databases (AWS RDS, Google Cloud SQL), data warehouses (BigQuery, Snowflake, Redshift), and key-value or document stores. Access is usually split between cloud IAM (who can manage the database) and the database's own users and permissions (who can query it), which makes visibility trickier.

**Block storage and snapshots.** The virtual disks attached to VMs, and **snapshots** (point-in-time copies of those disks). Snapshots are easy to forget, and copying or sharing a snapshot is a quiet way to take a whole disk's contents.

**Other places data hides:** file shares, container image registries (images can contain data), logs themselves, caches, and backups in a separate account.

## Controlling access to storage

Access to a bucket is decided by a combination of:

- **Identity policies:** what the requesting identity is allowed to do (Chapter 2).
- **Bucket (resource) policies:** a policy attached to the bucket itself saying who may access it. Resource policies can grant access to identities in *other* accounts, or even to everyone.
- **Organization guardrails:** for example, "no bucket in this organization may be public."
- **Access control lists:** an older, per-object permission system, best turned off.

The famous failure is the **public bucket**: a bucket whose policy grants read access to everyone on the internet. Many data leaks over the past decade were exactly this. Providers now block public access by default and offer organization-wide settings to prevent it. But it's still worth detecting any change that makes data public, because legitimate reasons are rare and the impact is immediate.

Two other access mechanisms to know:

- **Pre-signed URLs** (AWS) and **signed URLs** (GCP): temporary links that let anyone holding the link download a specific object, without credentials. Useful for sharing, and also a way for someone with access to hand data to someone without it, outside normal access controls.
- **Cross-account access:** a bucket policy that allows an identity in another account to read or write. Legitimate for data sharing and vendor integrations. Also a persistence and exfiltration technique.

## The logging gap

Recall the control plane vs data plane distinction from Chapter 0. For storage:

- **Control plane events** (create bucket, change bucket policy, delete bucket) are logged by default.
- **Data plane events** (read object, write object, list objects) are usually **not logged by default.** In AWS they're "data events" in CloudTrail and must be enabled per bucket or account, at extra cost. In GCP they're "Data Access" audit logs, which are off by default for most services.

This creates a dangerous blind spot. An attacker with read access to a bucket can download its entire contents, and if data events weren't enabled, the audit log may show nothing at all. During an incident, you'd be left unable to answer "what did they take?", which is often the most important question for legal and leadership.

The practical approach for D&R:

- **Enable data access logging on the buckets and databases that hold crown jewels,** whatever it costs. The volume is often modest for sensitive data that's accessed through a few known paths.
- **Accept gaps elsewhere,** consciously, and know where they are.
- **Use other signals as proxies** where data logging is missing: network egress volume, encryption key usage (below), and the bucket's own access logs.

## Encryption and key management

Almost all cloud data is encrypted at rest. That sounds like it should stop attackers, but it mostly doesn't, for a simple reason: **if the identity reading the data is allowed to read it, the cloud decrypts it for them automatically.** Encryption at rest protects against stolen physical disks and some provider-level risks. It doesn't protect against a stolen credential with read access.

Where encryption does help D&R is **customer-managed keys** in a **Key Management Service** (AWS KMS, Google Cloud KMS, Azure Key Vault):

- The data is encrypted with a key you control in KMS.
- To read the data, an identity needs permission on the data *and* permission to use the key.
- **Every use of the key is logged** in the audit log, even when data access itself isn't.

That last point is valuable. If a bucket holding sensitive data is encrypted with a dedicated key, then reading objects from it generates key-usage events (decrypt calls) in the audit log. Unusual decrypt volume, or decrypts by an identity that doesn't normally use the key, is a strong exfiltration signal that works even without full data logging. One caveat: some cost-saving features (such as S3 Bucket Keys) cache keys so that far fewer decrypt calls reach KMS, which weakens this signal. It's worth knowing whether they're turned on for the buckets you care about.

Keys also add a second lock: an attacker who gains access to the bucket but not the key can't read the data. Separating "who can read the bucket" from "who can use the key" is a cheap way to add depth around crown jewels.

A note on the vocabulary: you'll hear **envelope encryption**, which means data is encrypted with a data key, and the data key is itself encrypted with a master key in KMS. You don't need the details. The practical point is that the master key never leaves KMS, and its use is logged.

## How data leaves

Exfiltration in the cloud has more paths than on a traditional network. The main ones:

1. **Download through the API.** The attacker uses a stolen credential to read objects or query a database, and the data flows out over the internet to wherever they are. The most common path.
2. **Copy to an attacker-controlled account.** Rather than downloading, the attacker copies objects directly from your bucket into a bucket they own, provider-to-provider. This can be fast, may not cross your network at all, and may look like a normal copy operation in the logs.
3. **Share instead of copy.** Share a disk snapshot, database snapshot, or machine image with an external account. The attacker then makes their own copy at leisure. Very quiet, and very effective.
4. **Make it public.** Change a policy so the data is readable by anyone, then download it.
5. **Pre-signed URLs.** Generate links and download through them.
6. **Through a compromised workload.** Code running inside your environment (a pod, a VM) reads the data with its own legitimate identity and sends it out over the network. This is where network egress controls matter (Chapter 8).

High-value detections, therefore:

- Snapshots or images shared with accounts outside the organization.
- Bucket or database policies changed to allow external or public access.
- Copy or replication operations to destinations outside the organization.
- Unusual volume of reads or key decrypts on sensitive data, especially by an identity that doesn't normally read it.
- Large outbound network transfers from systems that hold sensitive data.

## Data perimeters

The strongest modern defense against most of these paths is a **data perimeter**: organization-level rules saying "our data may only be accessed by our identities, from our networks, and only moved to our resources."

- In AWS, this is built from organization policies (SCPs and resource control policies) and VPC endpoint policies, using conditions like "the resource must belong to our organization" or "the request must come from our network."
- In GCP, the main tool is **VPC Service Controls**, which draws a perimeter around projects and blocks data from moving across it, even for identities that would otherwise have access.

With a data perimeter in place, a stolen credential used from the attacker's own laptop fails (wrong network), and copying data to an attacker's bucket fails (wrong organization). It doesn't stop every path (a compromised workload inside the perimeter can still read data), but it removes many of the easiest ones. For a company protecting something like model weights, perimeters are a core control, and D&R should know exactly where they are and what they exclude.

## Questions to think about

1. During an incident, legal asks "did the attacker take customer data from bucket X?" Data events were not enabled. What other evidence could help you answer, and how confident could you be?
2. Why is sharing a disk snapshot with an external account a more attractive exfiltration path for a careful attacker than downloading files?
3. If you could only add one control around a bucket holding model checkpoints, would you choose data access logging, a customer-managed KMS key, or a data perimeter? Why?
