# Chapter 12 — Glossary and further reading

## Glossary

**APT (Advanced Persistent Threat):** a well-resourced, usually state-sponsored adversary that conducts long-term, targeted intrusions.

**ATT&CK (MITRE ATT&CK):** a public knowledge base of adversary tactics and techniques; the common vocabulary of D&R.

**Assume breach:** the premise that prevention will eventually fail, so you must be able to detect and respond to an attacker already inside.

**Backtest:** running a detection against historical data to see what it would have fired on before deploying it.

**Blast radius:** how much an attacker can reach from a given foothold.

**Blue team:** defenders. **Red team:** adversary emulators. **Purple team:** the two working together technique by technique.

**C2 (Command and Control):** the channel an attacker uses to communicate with and control compromised systems.

**Containment / Eradication / Recovery:** stopping the damage from spreading / removing all attacker access / restoring normal operation.

**Crown jewels:** the most valuable assets an organization must protect.

**Detection as code:** treating detections like software: version-controlled, reviewed, tested, deployed through a pipeline.

**Dwell time:** time an attacker is present before detection.

**EDR (Endpoint Detection & Response):** agent software on hosts that records process, file, and network activity and can take response actions.

**Exfiltration:** moving data out of the environment.

**IC (Incident Commander):** the person who owns coordination of an incident response.

**IdP (Identity Provider):** the system that authenticates users for SSO (e.g. Okta, Entra ID, Google Workspace).

**IOC (Indicator of Compromise):** an artifact like a hash, IP, or domain associated with malicious activity. Useful but brittle.

**Infostealer:** malware that harvests credentials, cookies, and tokens from a device.

**Lateral movement:** moving from one system or identity to another within an environment.

**Least privilege:** granting only the access needed for a role or task.

**Non-human identity:** service accounts, API keys, workload identities, CI credentials, and AI agents.

**OAuth consent abuse:** tricking a user or admin into granting a malicious app access to their data.

**Persistence:** mechanisms an attacker uses to keep access over time (keys, tokens, scheduled tasks, new accounts).

**Phishing-resistant MFA:** MFA (e.g. FIDO2/WebAuthn, passkeys) that is bound to the legitimate site and can't be relayed by a phishing proxy.

**Pivoting:** using one piece of evidence to find related evidence.

**Playbook / Runbook:** documented steps for responding to a type of alert or incident.

**Privilege escalation:** gaining more permissions than initially obtained.

**Pyramid of Pain:** model ranking indicators by how costly they are for an attacker to change, from hashes (cheap) to TTPs (expensive).

**Risk-based alerting (RBA):** accumulating risk scores per entity from many low-confidence signals, alerting when a threshold is crossed.

**SIEM (Security Information & Event Management):** a platform to collect, search, and run detections over security logs.

**Sigma:** an open, vendor-neutral format for writing detection rules.

**SOAR (Security Orchestration, Automation & Response):** tooling for automating triage and response workflows.

**SOC (Security Operations Center):** the function (or team) that monitors and responds to security alerts.

**Tabletop exercise:** a facilitated discussion of how the organization would respond to a hypothetical incident.

**Telemetry:** the logs and event data used for detection and investigation.

**Threat hunting:** proactive, hypothesis-driven searching for undetected malicious activity.

**Threat model:** a structured view of what you're protecting, from whom, and how they might attack.

**TTPs (Tactics, Techniques, Procedures):** how an adversary operates; the top of the Pyramid of Pain.

**Triage:** fast initial assessment of an alert to decide its disposition and urgency.

## Further reading

Chosen for "how to think" value, not tool-specific detail. All are widely known; check for the latest editions.

**Foundational concepts**

- *Intelligence-Driven Incident Response* — Scott J. Roberts & Rebekah Brown. Connects threat intelligence and IR into one way of thinking. Probably the single best book-length fit for this guide.
- *The Practice of Network Security Monitoring* — Richard Bejtlich. Older and network-focused, but the clearest explanation of the "collect, detect, analyze" mindset.
- NIST SP 800-61 (Computer Security Incident Handling Guide). The canonical incident lifecycle. Skim for structure.
- Google's *Site Reliability Engineering* book, chapters on incident management and postmortem culture. Not security-specific, but the best writing on running incidents and blameless postmortems.

**Detection engineering**

- Palantir's *Alerting and Detection Strategy Framework* (blog post and GitHub repo). Short and very practical.
- David Bianco's *The Pyramid of Pain* blog post. Ten minutes, lifelong value.
- John Lambert's "Defenders think in lists. Attackers think in graphs" post.
- SpecterOps blog posts on detection engineering and "capability abstraction" (how to reason about what a detection really covers).
- The *Detection Engineering Weekly* newsletter, for a feel of current practice.

**Adversaries and techniques**

- MITRE ATT&CK, especially the Cloud, SaaS, Identity Provider, and Containers matrices. Read tactic descriptions and a handful of technique pages.
- Public threat intelligence reports from major vendors and from government agencies (e.g. CISA advisories), focused on identity and cloud intrusions.
- Anthropic's own public threat intelligence reports on misuse of its models, for the AI-specific angle and the overlap with your background.

**AI-lab-specific**

- Anthropic's Responsible Scaling Policy and its published materials on ASL-3 security measures.
- RAND, *Securing AI Model Weights* (2024). The clearest public treatment of what it takes to protect frontier model weights against increasingly capable attackers.

**Incident write-ups**

Read public postmortems of real breaches (many companies publish them after identity, CI/CD, or SaaS-related incidents). For each, apply the questions from Chapter 11: initial access, where detection could have happened, what the containment trade-offs were, and what you would change.
