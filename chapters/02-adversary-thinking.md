# Chapter 2 — Thinking like the adversary

Detection is downstream of understanding the adversary. You can't decide what to look for until you know what you're defending, who wants it, and how they would plausibly get it. This chapter covers the mental models D&R people use for that.

## Start with what you're protecting

Threat modeling for D&R starts with **crown jewels**: the assets whose compromise would hurt most. For a typical tech company:

- Production systems and the data they hold (customer data especially).
- Source code and the build/release pipeline, because control of the pipeline means control of what ships.
- Credentials and signing keys: cloud root accounts, CI secrets, code-signing keys, identity provider admin.
- Executive and finance communications (fraud, M&A).
- For an AI lab, specifically: **model weights**, training infrastructure, unreleased research, and the systems that enforce model safety. (Chapter 10.)

Then ask the reverse question: **what are the paths to those assets?** Not in the abstract. Concretely, in *your* environment. Who has access to model weights? From which machines? Through which systems? Authenticated how? Every one of those paths is a route an attacker could take, and a place you need visibility.

This is sometimes called thinking in **attack paths** or **attack graphs**. A famous line from John Lambert of Microsoft sums it up: *"Defenders think in lists. Attackers think in graphs."* Defenders list assets and controls. Attackers see that the laptop of an engineer with production access, who reuses a personal password on a breached site, is one hop from the crown jewels. The best D&R engineers think in graphs too.

## Who are you defending against?

Rough categories, from least to most capable:

- **Opportunistic attackers:** automated scanning, credential stuffing, commodity malware, crypto-miners hunting exposed cloud keys. High volume, low sophistication, not targeting you specifically. Your abuse experience maps well here.
- **Financially motivated criminal groups:** ransomware operators, data extortion crews, business email compromise. Often skilled, often run like businesses, increasingly using social engineering against help desks and identity systems. Groups in recent years have shown how far phone-based social engineering of IT support and MFA reset flows can go.
- **Insiders:** current or former employees or contractors, malicious or coerced. They already have access, so the "get in" step doesn't exist for them.
- **State-sponsored actors (APTs, "advanced persistent threats"):** intelligence services and their contractors. Patient, well-resourced, goal-driven. For an AI lab, this is the category that sets the bar.

Different adversaries justify different investments. A sound D&R strategy states which adversaries it is primarily designed against, and accepts that it may not catch the very best of them while still making their lives as hard as possible.

## The attack chain

Models of attacker progression go by different names (Lockheed Martin's Cyber Kill Chain, Mandiant's attack lifecycle, the tactic columns of MITRE ATT&CK). They agree on roughly the same flow:

1. **Reconnaissance:** learn about the target. Employees on LinkedIn, exposed services, leaked credentials, tech stack from job postings.
2. **Initial access:** get a first foothold. Phishing, stolen credentials or sessions, exploiting an internet-facing service, a malicious dependency, a compromised vendor.
3. **Execution:** run code or commands.
4. **Persistence:** make sure access survives reboots, password resets, and session revocation. New OAuth apps, extra MFA devices, SSH keys, cloud access keys, scheduled tasks, backdoored CI jobs.
5. **Privilege escalation:** get more rights than the initial foothold had.
6. **Defense evasion:** disable or avoid logging and security tools, blend in.
7. **Credential access:** harvest more credentials: browser cookies, cloud tokens in config files, password managers, memory.
8. **Discovery:** map the environment. What's here, who has access to what, where the valuable stuff is.
9. **Lateral movement:** move from system to system and identity to identity toward the target.
10. **Collection:** gather the data of interest.
11. **Command and control (C2):** maintain a communication channel with the attacker.
12. **Exfiltration:** get the data out.
13. **Impact:** encrypt, destroy, manipulate, or (for espionage) often nothing visible at all.

Real intrusions don't run neatly in order. Steps repeat (discover, move, escalate, discover again), and some get skipped. The value of the model is not its order. It gives you a **checklist of opportunities to detect**. For any given threat, you can ask: at which of these stages would we see it?

An important asymmetry:

- **Early stages (initial access) are noisy and varied.** There are endless phishing lures and exploit variants, and attackers change them constantly.
- **Middle stages (discovery, credential access, lateral movement, persistence) are constrained.** Once inside, attackers have to do certain things to make progress, and there are only so many ways to do them in a given environment. These behaviors are often the most reliable detection points, because the attacker can't easily avoid them.

That's a deep idea worth internalizing: **detect what the attacker must do, not what they happen to do.** An attacker can choose any phishing domain. They can't reach your model weights without, at some point, authenticating to something that has access to them.

## MITRE ATT&CK: the shared vocabulary

**MITRE ATT&CK** is a public knowledge base of adversary behavior, organized as **tactics** (the *why*: persistence, lateral movement, exfiltration) and **techniques** (the *how*: e.g. "Steal Web Session Cookie," "Valid Accounts: Cloud Accounts," "Account Manipulation: Additional Cloud Credentials"). There are matrices for enterprise (Windows, macOS, Linux, cloud, SaaS, identity providers, containers), mobile, and ICS.

How D&R teams use it:

- **As vocabulary.** "We have good coverage on T1078 for our IdP but nothing for T1550 in the cloud plane" is a sentence a D&R team can act on.
- **For coverage mapping.** Lay your detections over the matrix to see where you're blind.
- **For threat intel.** Reports on a threat actor list the techniques they've used, which you can compare to your coverage.

And its limits, which you should be able to state, since naive use is common:

- **Coverage of a technique is not binary.** One detection for "PowerShell" doesn't cover every malicious use of PowerShell. A heat map full of green cells can be badly misleading.
- **Not all techniques matter equally to you.** Prioritize by your threat model and environment, not by filling the matrix.
- **It describes techniques, not your environment.** The specific way lateral movement looks in *your* Kubernetes-plus-SSO-plus-cloud setup is something you have to work out.

Being able to criticize a "we cover 85% of ATT&CK" claim is a good signal of maturity in an interview.

## The Pyramid of Pain

David Bianco's **Pyramid of Pain** ranks indicators by how much it hurts the attacker when you detect on them. From bottom (easy for the attacker to change) to top (painful):

1. **Hash values:** change one byte of the file, new hash.
2. **IP addresses:** trivially rotated.
3. **Domain names:** cheap to replace.
4. **Network/host artifacts:** a specific user-agent, a registry key, a file path. Some effort to change.
5. **Tools:** the attacker has to find or build a different tool.
6. **TTPs (tactics, techniques, procedures):** the attacker has to change *how they operate*. Most painful.

You know this from abuse already: blocking IPs or email domains works for a day. Blocking a *behavior* (signup-then-immediately-create-API-key-then-burst) holds up much longer. D&R uses the same reasoning: **indicators of compromise (IOCs)** like hashes and IPs are useful for quickly checking "were we hit by this known campaign?" but durable detection is built on **behaviors**.

## Threat intelligence: useful in proportion to its specificity

Threat intel ranges from "nation-states are targeting AI companies" (strategic, broadly true, not actionable by itself) to "this actor uses this specific technique to steal session cookies from this browser and replays them from residential proxies" (tactical, actionable).

How to judge threat intel:

- **Does it change a decision?** If a report doesn't change what you detect, prioritize, or harden, it's background reading.
- **Is it about behavior or just IOCs?** IOCs from a report a month old are often stale.
- **What's the source and confidence?** Vendor intel has commercial incentives. Government advisories can be authoritative but delayed.
- **Is it relevant to our environment?** A great Windows domain-controller technique matters little to a company that runs on macOS laptops and cloud identity.

## Assume breach, operationally

"Assume breach" sounds like a slogan, but it implies specific questions you should ask regularly:

- If an attacker had **an engineer's laptop** right now, what could they reach, and would we notice?
- If an attacker had **a valid SSO session** for a random employee, what would we see?
- If an attacker had **a cloud access key** from a CI job, what could they do, and which logs would show it?
- If an **insider** wanted to copy something sensitive, how would they do it, and would anything fire?

Each of these is a scenario you can reason through (and later test, Chapter 7). Writing these down for your environment is one of the highest-value things a D&R engineer can do, and it's a skill that maps directly onto the abuse instinct of "if I were the attacker, how would I get around this?"

## Questions to think about

1. Pick an asset you protect today. Sketch, in your head, the attack graph to reach it. Which hop is easiest for an attacker? Which hop is easiest for you to watch?
2. In account takeover, what does the attacker *have* to do after login to get value? Is that a better detection point than the login itself?
3. Someone says "our detections cover 85% of MITRE ATT&CK." What three questions would you ask next?
