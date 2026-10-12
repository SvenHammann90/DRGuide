# Chapter 8 — Modern attack surface: identity, cloud, supply chain, insiders

Classic security training focuses on a corporate network: Windows domain, file servers, firewalls. Modern tech companies, AI labs included, look different: cloud infrastructure, SSO into dozens of SaaS apps, macOS or Linux laptops, code hosted on GitHub, everything deployed through CI/CD, and remote employees everywhere. The attack surface has moved, and so have D&R priorities.

This chapter covers the four areas that matter most now. The first is where your background is most directly valuable.

## 1. Identity is the perimeter

In a cloud-first company there's barely a network to break into. **Identity is the control plane.** If you hold a valid identity (a session, a token, a key) you can reach whatever that identity can reach, from anywhere. So most modern intrusions are, at their core, identity compromises.

You already know the consumer-side version of every one of these. Here's how they look against a workforce:

**Phishing and credential theft.** Still common. Increasingly with **adversary-in-the-middle (AiTM) phishing kits** that proxy the real login page, capturing the password *and* the session cookie after MFA, which defeats SMS, TOTP, and push MFA. Only phishing-resistant MFA (FIDO2/WebAuthn hardware keys or passkeys, which bind authentication to the real domain) stops this class. If an organization has rolled out hardware keys everywhere, the attacker moves to other techniques below.

**Session and token theft.** Rather than steal a password, steal what you get *after* login: browser session cookies (via infostealer malware on the device), OAuth refresh tokens, cloud CLI credentials sitting in config files, personal access tokens for code hosts. These bypass MFA entirely because authentication already happened. Detection angles: session used from a different device/IP/ASN than it was issued to, impossible travel within a session, token use from unexpected infrastructure. Mitigations: short session lifetimes, device-bound sessions, re-authentication for sensitive actions.

**Help desk and recovery social engineering.** Attackers call IT support pretending to be an employee who lost their phone, and get MFA reset or a new device enrolled. Groups in recent years have run high-profile intrusions largely this way. Your account recovery experience translates directly: the recovery flow is the weakest link, and the signal is in the sequence (reset → new enrollment → login from new context → sensitive access).

**MFA fatigue.** Spamming push notifications until the user approves one. Mitigated by number matching and phishing-resistant MFA; detected by bursts of denied pushes followed by an approval.

**OAuth consent abuse.** Tricking a user (or admin) into granting a malicious third-party app access to mail, files, or code. The app then has durable access that survives password resets. A persistence technique your consumer background may not have covered but will grasp immediately.

**IdP compromise itself.** If an attacker gets admin on the identity provider, they can mint access to everything: add themselves to groups, add a federation trust, disable MFA for an account. Admin actions on the IdP are among the highest-value things to monitor in the whole environment.

**Non-human identities.** Service accounts, CI credentials, API keys, cloud roles assumed by workloads. Often more numerous than human identities, often over-privileged, rarely protected by MFA, and frequently leaked (committed to repos, printed in logs, left in container images). Many cloud intrusions start with a leaked key, not a phished human.

What transfers from your background: thinking about the full identity lifecycle (creation, authentication, session, recovery, privilege changes, deactivation), knowing that the post-authentication session is where much modern abuse happens, and pattern-matching on sequences of identity events. What's new: workforce identity has **privilege** in a way consumer accounts mostly don't. One compromised engineer may have production access. That makes the stakes and the detection priorities different: **you care most about the identities closest to the crown jewels.**

## 2. Cloud infrastructure

Cloud environments (AWS, GCP, Azure) are administered entirely through APIs, which is good news for D&R: almost everything is logged in the control plane audit log, if enabled. Typical attacker patterns:

- **Credential access:** stealing keys from developer machines, CI, instance metadata services (via SSRF bugs in web apps), or code repositories.
- **Discovery:** enumerating what the stolen identity can see: list buckets, list roles, describe instances. A burst of read-only "list/describe/get" calls from an identity that normally doesn't do that is a classic signal (though noisy, because engineers and tooling also do it).
- **Privilege escalation:** cloud IAM is complex, and there are many known paths where one permission lets you grant yourself more (e.g. being able to modify a role's policy, or pass a privileged role to a compute resource you control).
- **Persistence:** create new access keys, new users, new roles with trust to an attacker-controlled account, or modify existing trust policies.
- **Defense evasion:** stop or alter audit logging, delete log storage.
- **Exfiltration:** copy data from storage, snapshot disks and share them to an external account, make a bucket public.
- **Impact:** crypto-mining on stolen compute (the most common opportunistic outcome), or deletion/ransom.

**Kubernetes** adds another layer: compromised pods, privileged containers that let attackers escape to the node, service account tokens mounted into pods, overly broad cluster roles. Container runtime telemetry and the K8s API audit log are the main sources.

D&R judgment in the cloud mostly means knowing **which actions are rare and sensitive** in your environment (IAM changes, logging changes, cross-account trust, snapshot sharing), making sure those are logged and watched, and knowing which identities and accounts hold the crown jewels.

## 3. Software supply chain

Your company runs code it didn't write (open-source dependencies, vendor software, CI actions) and ships code through pipelines that, if compromised, let an attacker inject whatever they want. Key scenarios:

- **Malicious or compromised dependencies:** typosquatted packages, hijacked maintainer accounts publishing malicious versions, malicious install scripts. Well-known incidents have included backdoors planted in widely used libraries through long-running social engineering of maintainers.
- **Compromised CI/CD:** an attacker who can modify a pipeline (or a third-party action it uses) can steal every secret available to it and alter build artifacts. CI systems are rich targets because they hold powerful credentials.
- **Compromised vendors:** a SaaS provider or software vendor you trust is itself breached, and their access to you becomes the attacker's.
- **Developer environments:** malicious IDE extensions, compromised developer tools, poisoned dev containers.

For D&R this means: monitoring changes to CI configuration and secrets, watching for unexpected network egress from build systems, knowing which third parties have what access, and having a playbook for "vendor X announced a breach, what do we do?" (usually: find all integrations, rotate credentials, review their access logs).

## 4. Insider threat

Insiders already have legitimate access, so most perimeter and authentication detections don't help. Insider risk includes:

- **Malicious insiders:** stealing IP for a competitor or foreign government, or sabotage.
- **Departing employees** taking data with them (often the most common real-world case).
- **Coerced insiders:** pressured or recruited by external actors. A real concern for organizations holding nationally significant technology.
- **Negligent insiders:** not malicious, but careless with data or credentials.

Detection relies on **data-access telemetry and context**: who accessed what, compared to their role and history; bulk downloads; access to unusual data shortly before leaving; copying to personal storage or devices. HR context (resignation notice, performance issues) matters a great deal.

Insider work is also where D&R is most sensitive:

- **Privacy and trust.** Monitoring employees has real costs to culture and legal boundaries that vary by country. Good programs are transparent about what is monitored and why, collect what's proportionate, and restrict who can look.
- **Process.** Insider investigations usually involve HR and legal from the start, a small need-to-know group, and careful handling to avoid both tipping off the subject and wrongly accusing an innocent colleague.
- **Prevention is stronger than detection here.** Least privilege, separation of duties, multi-party approval for the most sensitive actions (e.g. two people required to access model weights) limit what any single insider can do. D&R should advocate for these.

You may find your abuse background helps here too: distinguishing "unusual but legitimate" from "malicious" in a population of mostly-legitimate actors is exactly the problem.

## The common thread

Across all four areas the same idea keeps appearing: **the attacker acts through legitimate access** — a real session, a real key, a real pipeline, a real employee. Very little of modern intrusion involves "malware" in the classic sense. That's why D&R today is so much about **identity, context, and behavior**, and why someone who has spent years asking "is this the real account holder doing this?" is better prepared than they might think.

## Questions to think about

1. An engineer's laptop is infected with an infostealer. List everything an attacker might now hold (cookies, tokens, keys) and where each could be used. Which would you revoke first?
2. Map three account takeover techniques you know from consumer products onto their workforce equivalents. Where are the workforce versions *more* dangerous?
3. What's the cost of watching employees too closely? How would you argue for an insider risk detection while respecting that cost?
