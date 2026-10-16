# Chapter 10 — Cloud logging and detection

This chapter pulls the earlier ones together from the detection side: which logs exist, what they contain, where they're blind, and which behaviors are worth detecting. It builds on the first guide's chapters on telemetry and detection engineering, applied specifically to the cloud.

## The audit logs

Each provider has a main audit log for its control plane:

**AWS CloudTrail.** Records API calls across AWS services.
- **Management events** (creating, changing, deleting resources and permissions) are recorded by default and viewable for 90 days in the console. To keep them longer and analyze them, you create a **trail** that sends them to storage, ideally an **organization trail** covering every account and region, delivered to a separate security account.
- **Data events** (reading and writing objects, invoking functions, and similar) are off by default and must be enabled selectively, at extra cost.

**Google Cloud Audit Logs.** Several types:
- **Admin Activity** logs (configuration changes) are always on and can't be turned off.
- **Data Access** logs (reading data and metadata) are off by default for most services and must be enabled. (BigQuery is a notable exception, logged by default.)
- **System Event** logs (Google-initiated changes) and **Policy Denied** logs (requests blocked by guardrails such as VPC Service Controls).

**Azure.** The Activity Log covers resource management. **Entra ID** sign-in and audit logs cover identity. Resource-specific diagnostic logs cover data access, and each must be enabled.

**Kubernetes audit logs.** Every request to the API server (Chapter 6). In managed clusters, usually exported to the provider's logging service once enabled.

## Reading an audit log entry

Across providers, an audit log entry answers roughly the same questions. Knowing them is enough to read any provider's logs with AI help:

- **Who:** the principal that made the call. For a role session, this includes the role and a session name, and sometimes a source identity tracing back to a human.
- **What:** the action (for example, "create access key," "get object," "set IAM policy").
- **On what:** the resource affected.
- **From where:** source IP address, and sometimes whether it came through a private endpoint.
- **With what:** the user agent, meaning the tool or library that made the call.
- **Result:** success, or the error. **Access denied errors are themselves useful signals.**
- **When:** timestamp.
- **Context:** request parameters, and whether MFA was used.

Most investigation questions come down to filtering and joining on those fields: "everything this principal did," "everything from this IP," "every change to this resource," "every access-denied error in the last hour."

## The identity chain problem, again

The hardest part of cloud log analysis is usually **attribution**: working out which human or system is really behind an action. A single human action might appear as:

1. An SSO login in the identity provider's logs (Alice, from her laptop).
2. A role assumption in the cloud audit log (a session for role `engineer` with session name `alice@company`).
3. A second role assumption into another account (role `prod-admin`, now possibly with a different session name).
4. The actual action, recorded under `prod-admin`.

Each hop may record slightly different identifiers. Providers have features to make this traceable (AWS "source identity," consistent session names enforced by SSO, GCP's record of service account impersonation chains), and good D&R teams make sure they're used. But you should expect to spend real effort, and real data engineering, on stitching identity chains together. It's the cloud version of tying a consumer session back to the account and device that created it.

## Native threat detection services

Providers offer built-in detection services:

- **AWS GuardDuty:** analyzes CloudTrail, VPC flow logs, DNS logs, and Kubernetes audit logs, and produces findings like "instance credentials used from an external IP," "API calls from a known malicious IP," "unusual crypto-mining activity," or "root user activity."
- **Google Security Command Center:** threat detection and misconfiguration findings across GCP.
- **Microsoft Defender for Cloud:** similar for Azure, plus some multi-cloud support.

Plus a large commercial market: **CSPM** (cloud security posture management, which finds misconfigurations), **CDR** (cloud detection and response, which finds active threats), **CWPP** (cloud workload protection, meaning runtime agents for VMs and containers), and **CNAPP**, which bundles all of the above.

How a D&R engineer should think about these:

- **They're a good baseline,** especially for known, generic attack patterns. Turn them on.
- **They don't know your environment.** They can't know that a particular role should only ever be used from your CI, or that a particular bucket holds model weights. Your most valuable detections will be ones you write, using context only you have.
- **Posture vs detection.** A misconfiguration finding ("this bucket is public") is a prevention problem. A threat finding ("this credential is being used from an unusual place") is a detection problem. Both matter, but they go to different queues and need different responses.

## A catalogue of behaviors worth detecting

Here are cloud behaviors that tend to make good detections, grouped by the attack stages from the first guide. Each is a behavior, not a specific tool, so it holds up as attackers change tools. For each, think about what's normal in your environment, because that decides whether it's an alert, an enrichment, or a hunt.

**Initial access and credential use**
- A long-lived key used for the first time in a long time, or from a new network.
- Machine or workload credentials used from outside the machine or cluster they belong to.
- Pipeline credentials used from outside the CI provider's networks.
- Console logins without MFA, or from unusual locations, especially to privileged roles.
- Any root user activity.

**Discovery**
- A burst of list and describe calls across many services from one identity.
- A spike in access-denied errors from one identity.
- `GetCallerIdentity` (or the equivalent) from an unusual network, especially as the first call of a session.

**Privilege escalation**
- An identity modifying its own permissions.
- Creating access keys or login profiles for another user.
- Attaching powerful policies (admin, or wildcard permissions).
- Passing privileged roles to new compute resources or functions.
- Granting service account impersonation rights (GCP).
- New ClusterRoleBindings to powerful roles in Kubernetes.

**Persistence**
- New IAM users, new access keys, or new roles, in accounts where these normally come only from IaC.
- Changes to role trust policies, especially trust to external accounts or identity providers.
- New identity providers or federation trusts configured.
- New Kubernetes DaemonSets or CronJobs outside the pipeline.
- New functions, or changes to function code, outside the pipeline.

**Defense evasion**
- Stopping, deleting, or modifying audit log trails or log sinks.
- Disabling threat detection services.
- Changing guardrails or organization policies.
- Activity in regions you don't use.

**Collection and exfiltration**
- Snapshots, images, or databases shared with external accounts.
- Bucket policies changed to allow public or external access.
- Unusual volume of object reads, or KMS decrypts, on sensitive data.
- Data copy or replication to destinations outside the organization.
- Large outbound transfers from systems holding sensitive data.

**Impact**
- Mass deletion of resources, snapshots, or backups.
- Sudden creation of many large compute instances (often crypto-mining), especially GPU instances.
- Encryption key deletion scheduled, or key policies changed to lock out the owner.

Notice how many of these share a pattern: **a sensitive action, by an identity that doesn't normally perform it, outside the path it normally comes through.** That pattern, and the context needed to evaluate it (what's normal for this identity, which path is the approved one), is the core of cloud detection.

## Making cloud detection work in practice

A few practical lessons that come up again and again:

- **Centralize logs in a separate, locked-down account.** If an attacker compromises a production account, they shouldn't be able to delete its logs. Write-once storage for logs is common practice.
- **Know your normal paths.** Write down, for each sensitive action, which identity normally performs it and through which pipeline. Detections then become "this action, not through that path."
- **Invest in context.** Account inventory (what is this account for, who owns it?), resource tags (is this bucket sensitive?), and identity mapping (who is behind this role session?) make every alert better.
- **Expect to tune.** Cloud environments are full of automation doing unusual-looking things. Your first version of almost any detection will be noisy.
- **Watch for silent failure.** A trail that stops delivering, a log sink that breaks, a new account that never got logging enabled. These are the most dangerous gaps because everything looks quiet.

## Questions to think about

1. Pick three detections from the catalogue above. For each, what would you need to know about your environment to make it precise enough to page someone?
2. Why should audit logs be stored in a separate account? What attack does that protect against, and what wouldn't it protect against?
3. A native detection service flags "API calls from a known malicious IP" for a role used by your data pipeline. What would you check to decide whether this is a real compromise or a false positive?
