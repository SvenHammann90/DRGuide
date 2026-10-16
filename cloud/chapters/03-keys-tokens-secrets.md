# Chapter 3 — Keys, tokens, and secrets

"Cloud keys" come up constantly in D&R because they're the most common way attackers get into cloud environments. The word "key" is overloaded, though, and the mental model gets much clearer once you separate the meanings. This chapter builds that model: what kinds of credentials exist, how long they last, where they live, how they leak, and why revoking them is trickier than it looks.

## First, untangle the word "key"

In cloud conversations, "key" can mean at least four different things:

1. **Access credentials:** something that proves identity to the cloud API. "An AWS access key," "a GCP service account key." This is what people usually mean by "cloud keys," and it's the main subject of this chapter.
2. **Encryption keys:** keys used to encrypt and decrypt data, usually managed by a **Key Management Service (KMS)**. They don't log you into anything. They control whether data can be read. (Chapter 7.)
3. **SSH keys:** key pairs used to log into machines over SSH.
4. **Signing keys:** keys that prove something is authentic, like code-signing keys, or the keys an identity provider uses to sign login tokens. If an attacker steals an identity provider's signing key, they can forge logins for anyone. This is rare but catastrophic.

When someone says "a key leaked," your first question should be: which kind? The response is completely different for each.

## Access credentials: the two families

All access credentials answer the question "who are you?" for the cloud API. They split along one dimension that matters more than any other: **how long they live.**

### Long-lived credentials

These are valid until someone deletes them. Examples:

- **AWS access keys** attached to IAM users. They come as a pair: an access key ID (starting with `AKIA`) and a secret access key. Anyone who holds both can act as that user, from anywhere, until the key is deactivated.
- **GCP service account keys:** JSON files containing a private key for a service account. By default they never expire.
- **Azure service principal secrets** and certificates.
- **API keys and personal access tokens** for SaaS and developer tools (code hosts, package registries, monitoring services).

Long-lived credentials are convenient (create once, paste into a config, forget) and that's exactly the problem. They get copied into config files, environment variables, scripts, CI settings, chat messages, and documentation. They outlive the projects they were made for. Nobody remembers which systems depend on them, so nobody dares delete them.

**Most cloud breaches that start with a leaked credential start with a long-lived one.**

### Short-lived credentials

These expire on their own, usually within minutes to hours. Examples:

- **AWS temporary credentials** from assuming a role, via the Security Token Service (STS). The access key ID starts with `ASIA`, and they come with a session token. Typical lifetime: one to twelve hours.
- **GCP OAuth access tokens**, typically valid for one hour.
- **Azure tokens** for managed identities.
- **Kubernetes projected service account tokens** (Chapter 5).

Short-lived credentials are issued by something that has already verified identity: SSO login, a machine's attached role, a CI system's identity. They're still dangerous if stolen, but the window is small, and they don't accumulate in old config files.

The direction of travel in cloud security is clear: **replace long-lived credentials with short-lived ones everywhere possible.** Many organizations block creating new long-lived keys entirely with a guardrail (Chapter 1).

A small but useful detail: in AWS, the key prefix tells you which family you're looking at. `AKIA` means long-lived user key. `ASIA` means temporary session credentials. Seeing an `AKIA` key used by a production workload is often a finding in itself.

## Keyless: how workloads get credentials without secrets

If long-lived keys are bad, how does code get credentials? The answer is that the platform it runs on vouches for it:

- **Machines get credentials from the metadata service.** A VM with an attached role can ask a special local address (the **instance metadata service**) for temporary credentials, and gets fresh ones automatically. No secret is stored on disk. (Chapter 4 covers how attackers abuse this.)
- **Kubernetes pods get credentials through workload identity.** The cluster gives each pod a signed token saying "I am pod X in namespace Y with service account Z," and the cloud exchanges that for cloud credentials (Chapter 5).
- **CI pipelines get credentials through OIDC federation.** A CI system like GitHub Actions can issue a signed token saying "this job is running from repository R, branch B." The cloud is configured to trust tokens from that CI system, under conditions, and exchanges them for temporary credentials. No cloud secret is stored in the CI system at all. (Chapter 9.)

The pattern is the same each time: **identity comes from where the code runs, attested by a trusted issuer, exchanged for short-lived cloud credentials.** It's a big improvement. It also moves the risk: now the important question is *who can run code in that place*, and *is the trust condition specific enough?* A CI trust policy that accepts tokens from any repository, instead of one specific repository, lets anyone with a repository on that CI platform assume your role. This misconfiguration has been found in real environments.

## Where credentials live

When an attacker compromises something, they go looking for credentials. When you scope an incident, you need to ask the same question: what credentials could have been taken from here? Common places:

**On engineers' laptops:**
- Cloud CLI configuration files (`~/.aws/credentials`, `~/.config/gcloud/`, `~/.azure/`), which may hold long-lived keys or cached short-lived tokens and refresh tokens.
- Kubernetes config files (`~/.kube/config`) with cluster credentials.
- Browser cookies for the cloud console and SSO sessions.
- Environment variables, shell history, `.env` files in project folders.
- Password managers and SSH keys.

**On servers and containers:**
- Credentials available from the instance metadata service.
- Environment variables set at deploy time.
- Kubernetes secrets mounted as files.
- Service account tokens mounted into pods.
- Config files baked into container images (a recurring mistake).

**In delivery systems:**
- CI/CD secret stores and variables.
- Build logs that accidentally print secrets.
- Infrastructure-as-code state files (Terraform state often contains secrets in plain text).

**In shared places:**
- Code repositories, including history: a key committed and then deleted is still in the git history.
- Chat messages, tickets, wikis, and documents.
- Public paste sites and public repositories, when something goes badly wrong.

This list is also the reason **infostealer malware** is so dangerous to cloud environments (first guide, Chapter 8). One infected developer laptop can yield cloud keys, kube configs, CI tokens, and SSO cookies all at once.

## Secret managers

The right home for a secret that has to exist is a **secret manager** (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, and similar). Code fetches the secret at runtime, using its workload identity, instead of having it written into config.

Benefits for D&R:

- **Every read is logged.** You can see which identity fetched which secret, when. "A secret was read by an identity that has never read it before" is a solid detection.
- **Rotation is possible.** Changing a secret in one place updates it everywhere.
- **Fewer copies.** The secret isn't scattered across config files and images.

Secret managers don't solve everything. Code that fetches a secret still holds it in memory, and an attacker who controls that code can read it. But they turn an invisible sprawl into a logged, controllable system.

## How credentials leak, and how fast

Common leak paths, roughly in order of how often they come up:

1. **Committed to a code repository,** especially a public one, or a private one that later becomes public.
2. **Stolen from a developer machine** by malware or a compromised tool.
3. **Exposed by an application vulnerability,** typically by tricking a server into fetching its own metadata credentials (Chapter 4).
4. **Leaked through CI/CD:** printed in build logs, exposed to untrusted code (for example, pull requests from forks), or stolen through a compromised third-party build step.
5. **Shared carelessly:** pasted into chat, tickets, or documents.
6. **Left in artifacts:** container images, compiled binaries, backups.

On speed: automated scanners watch public code hosts continuously. Keys pushed to a public repository are commonly found and tried by attackers within minutes. Providers and code hosts run their own scanning too. GitHub, for example, scans public repositories for known credential formats and notifies providers, and AWS can automatically attach a restrictive quarantine policy to a leaked key. These help, but they're a safety net, not a plan.

## What attackers do with a stolen credential

The first moves are predictable, which makes them detectable:

1. **"Who am I?"** In AWS, an attacker calls `GetCallerIdentity`, which tells them which account and identity the key belongs to. It needs no permissions and is almost always the first call. Legitimate systems call it too, so it's not an alert on its own, but **a first-ever `GetCallerIdentity` from an unfamiliar network, using an old key, is a strong signal.**
2. **"What can I do?"** Enumeration: list buckets, list roles, list instances, try many API calls to see which succeed. This creates a burst of read calls, often many of them **failing with access denied errors**. A spike in access-denied errors from one identity is a classic sign of someone exploring with a stolen credential.
3. **Escalate and persist.** Create new keys, new users, new roles, or modify trust policies, so that access survives the original key being revoked.
4. **Act on objectives.** Mine crypto (opportunistic attackers), steal data, or move toward specific targets.

What helps you distinguish attacker from legitimate use:

- **Source network:** is the key being used from the place its workload normally runs (a known CI provider, your own cloud's IP ranges), or from a residential proxy, VPN service, or hosting provider you don't use?
- **User agent:** the tools used to make the calls. A key that normally comes from your deployment tool suddenly being used by a different CLI or scripting library is notable.
- **Behavior:** does the activity match what the identity normally does? A deploy key that normally updates services, suddenly listing every bucket, is off-pattern.
- **Timing:** first use after months of dormancy, or use right after a known leak event.

## Revoking credentials: harder than it sounds

When a credential is compromised, the instinct is "delete the key." That's necessary but often not sufficient, for three reasons:

**1. Credentials can spawn other credentials.** A long-lived key can be used to request temporary credentials, or to assume roles. Those temporary credentials don't automatically die when the original key is deleted. They stay valid until they expire. Providers offer ways to deal with this (in AWS, for example, you can attach a policy to a role that denies all sessions issued before a certain time, which is what the "revoke active sessions" button does), but you have to know to do it.

**2. The attacker may have created persistence.** New keys, new users, new roles, new trust relationships, new OAuth grants. Deleting the original key does nothing to these. This is why scoping (first guide, Chapter 5) matters: you need the full list of what the compromised identity did.

**3. Revoking can break production.** If a long-lived key is used by a production system, deleting it may cause an outage. Responders often have to decide between "revoke now and accept the outage" and "contain it another way while we rotate safely." A middle path is often to restrict the key (attach a deny policy, or limit it by source network) rather than delete it outright, which also preserves evidence.

A good mental checklist when a credential is compromised:

1. **Identify** exactly which credential, which identity, which account.
2. **Contain:** disable or restrict the credential, and revoke any sessions issued from it.
3. **Scope:** review everything that identity did since the earliest possible compromise time. Look for created credentials, users, roles, policy changes, and data access.
4. **Eradicate:** remove any persistence found.
5. **Rotate** anything else the attacker could have seen (other secrets in the same place the key leaked from).
6. **Fix the root cause:** why was a long-lived key there in the first place? Can it be replaced with keyless access?

## Questions to think about

1. A key starting with `AKIA` shows up in a public repository. Walk through what you'd want to know in the first ten minutes, and in what order.
2. Why doesn't deleting a compromised long-lived key end the incident? Name two things that could still let the attacker in.
3. If you replaced all long-lived keys with keyless access, where would attackers look next? What would you then need to watch?
