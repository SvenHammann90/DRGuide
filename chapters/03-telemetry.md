# Chapter 3 — Telemetry: you can only detect what you can see

Every detection, investigation, and hunt rests on one thing: data about what happened. If the log doesn't exist, isn't collected, isn't kept long enough, or can't be searched, then nothing else matters. That makes telemetry the least glamorous and most important part of D&R.

A senior D&R engineer's first question about any threat is not "what's the detection?" It's **"what would we even see?"**

## The main sources

You don't need to know every field of every log, but you need a map of which sources exist, what each tells you, and where each is blind.

**Identity provider (IdP) and SSO logs.** Okta, Entra ID (Azure AD), Google Workspace, and similar. Who authenticated, from where, with which factor, to which application. Session creation, MFA enrollment and reset, admin actions, policy changes. In a modern cloud-first company this is often the **single most important source**, because identity is the main control plane. Your account security background lives here.

**Cloud control plane audit logs.** AWS CloudTrail, GCP Cloud Audit Logs, Azure Activity Log. Every API call against the cloud: who created this instance, changed this IAM policy, read this secret, made this bucket public. Watch out for the difference between **management events** (changes to configuration, usually logged by default) and **data events** (e.g. reading objects from storage, often *not* logged by default because of volume and cost). Many exfiltration scenarios live in data events.

**Endpoint telemetry (EDR).** Endpoint Detection & Response agents on laptops and servers (CrowdStrike, SentinelOne, Microsoft Defender, osquery-based setups, etc.). Process creation with full command lines and parent processes, file writes, network connections, persistence mechanisms. This is where you see malware, hands-on-keyboard attackers, and suspicious tools. On developer laptops it's also very noisy, because developers run everything.

**Kubernetes and container telemetry.** K8s API server audit logs (who did what to the cluster), and runtime telemetry inside containers (process execution, unexpected shells, network calls). Important wherever production runs on containers, which is most places.

**SaaS application logs.** Email, chat, docs, code hosting (GitHub/GitLab audit logs), ticketing, HR systems. Who shared which document externally, who installed which third-party app, who cloned which repository, who created a personal access token.

**CI/CD and source control.** Pipeline runs, who changed a workflow file, which secrets a job accessed, what got deployed. The software supply chain (Chapter 8) is visible here or nowhere.

**Network telemetry.** DNS queries, proxy logs, VPC flow logs, firewall logs. Less central than it used to be (most traffic is encrypted, and "the network perimeter" has largely dissolved), but still valuable: DNS and egress logs are often how you spot command-and-control or exfiltration.

**Application logs for your own systems.** The internal tools and production services your company builds. Often the least standardized and most valuable for detecting abuse of internal privileges, such as an employee querying customer data they have no business reason to look at.

## Properties that matter more than the source list

For each source, a D&R engineer asks:

- **Coverage.** Is it on everywhere? EDR on 92% of laptops means 8% of laptops are dark, and an attacker might be on one of them. Is the cloud audit log enabled in *every* account and region, including the one the platform team spun up last month?
- **Fidelity.** Does it record what you need? A process log without command-line arguments is far less useful. A login log without the session ID can't be tied to later actions.
- **Latency.** How long from event to searchable? Minutes is fine for most things. Hours hurts during an active incident.
- **Retention.** How far back can you search? Many intrusions are discovered weeks or months after they began. If you only keep 30 days of logs, you may never be able to answer "when did this start?" Retention is often a cost negotiation, and D&R needs to argue for it.
- **Integrity.** Can an attacker who compromises a system tamper with or delete its logs? Good architectures ship logs off-host quickly to a place the attacker's foothold can't modify. "Log deleted" or "logging disabled" is itself a high-value detection.
- **Joinability.** Can you link events across sources? The holy grail is following one actor from IdP login → session → cloud API calls → data access. That requires shared identifiers (user, session, device, IP) and consistent normalization. In practice, **identity correlation across sources** is where much D&R engineering effort goes.

## Telemetry is a data engineering problem

This is where full-stack D&R roles differ most from analyst roles. Somebody has to:

- Get logs out of dozens of vendors' APIs reliably (and notice when a feed silently stops — **a dead log source is one of the most dangerous failure modes in D&R**, because everything looks quiet).
- Parse and normalize them into consistent schemas.
- Store them somewhere queryable at a sane cost: a SIEM (Splunk, Elastic, Chronicle/Google SecOps, Sentinel, Panther, and others) or a data lake/warehouse (Snowflake, BigQuery, S3 + query engines). Many modern teams use a lake for scale and cost, with detection logic running on top.
- Enrich events with context: who is this user, what team, what is their normal location, what's this device's owner, is this IP a known VPN egress?

You will likely use AI to write a lot of the parsing and pipeline code. What you need to judge is the **design**: what to collect, at what fidelity, kept how long, joined how, and how you'll know when it breaks.

## Context is half of telemetry

Raw events are not enough. "User X ran `aws s3 sync` on bucket Y" means nothing until you know:

- Is X an engineer whose job involves bucket Y?
- Is this their usual device? Their usual time of day?
- Is Y a public dataset bucket or the model-weights bucket?

So D&R depends on **context sources**: HR data (team, manager, role, start date, *termination date*), asset inventory (who owns this device, what is this host for), cloud resource tags and ownership, access-control data (who *should* have access to what). Often these context sources are incomplete, and maintaining them is a cross-team effort. A D&R engineer who can make context reliable makes every detection better.

You already know this pattern from abuse: an account's age, history, and linked entities often matter more than any single event. Same here.

## Thinking about cost honestly

Logging everything at full fidelity forever is impossibly expensive at scale. So D&R constantly makes trade-offs:

- **High-value, low-volume sources** (IdP, cloud management events, admin actions, code-host audit logs): keep everything, keep it long.
- **High-volume, mixed-value sources** (network flows, data-access events, endpoint file events): sample, filter, tier the storage (hot for recent, cold and cheap for older), or keep summaries.
- **Questions to ask** before cutting a source: which detections depend on it? Which investigation questions could we no longer answer? What would we lose in the incident where we need it most?

A good mental test: **imagine the post-incident review of a serious breach. Which log would you most regret not having?** Collect that one.

## The visibility map

A habit that pays off: keep a rough mental (or written) **visibility map** of your environment. For each major system and each major attack stage, do we have telemetry? Is it in the central platform? Is it searchable quickly? Has anyone tested that it actually shows the thing we think it shows?

When you hear about a new attack technique, the first move is to check it against that map: "Would we see this? In which log? Has anyone looked at what it actually looks like there?" Very often the honest answer is "we'd see part of it, in a source nobody has detections on." That's a useful finding in itself.

## Questions to think about

1. In your current role, which signal do you rely on most? What happens to your defenses if that signal silently stops flowing for a week? Would anyone notice?
2. An attacker uses a stolen session cookie to access a SaaS app. Which logs would show it, and what fields would let you distinguish the attacker from the real user?
3. If you had budget to keep only one extra source at 1-year retention, which would you choose for an AI lab, and why?
