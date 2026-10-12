# Chapter 6 — Incident response

An incident is when investigation turns into coordinated action. The technical work continues, but now it sits inside a structure: people with roles, decisions with trade-offs, communication with stakeholders, and an end state to reach. This chapter covers the parts of incident response (IR) that are about judgment and coordination rather than technique.

## What makes something an incident

Organizations define this differently, but a working definition: **an incident is a confirmed or strongly suspected security event that requires coordinated response beyond normal triage.** A single phishing email that nobody clicked is not an incident. A user who entered their password into the phishing page probably is. An attacker session active in production is definitely one.

Most organizations use **severity levels** (SEV1–4 or similar), based on:

- **Impact:** what's affected (crown jewels? customer data? one laptop?).
- **Confidence:** confirmed compromise or plausible suspicion?
- **Activity:** is the attacker still active?
- **Scope:** one system or many?

Severity determines who gets pulled in, how fast, and how often leadership is updated. A useful principle: **declare early, downgrade freely.** It's cheap to declare an incident and later say "false alarm." It's expensive to realize six hours in that you should have had a coordinated response from the start.

## Roles

Even small incidents benefit from explicit roles. Borrowed largely from emergency services' Incident Command System and from SRE practice:

- **Incident Commander (IC):** owns the overall response. Sets priorities, makes or escalates decisions, keeps people coordinated. The IC should usually *not* be the person deepest in the technical investigation, because the two jobs compete for attention.
- **Technical lead(s) / investigators:** do the analysis and the containment work.
- **Communications lead:** keeps stakeholders (leadership, legal, affected teams, sometimes customers) updated so investigators aren't interrupted.
- **Scribe:** maintains the timeline and decision log in real time.

In a small incident one person might hold several roles. The point is that someone *knows* they hold each one.

Things a good IC does that new responders often don't:

- Regularly states **current understanding, current priorities, and open questions** ("Here's what we know, here's what we're doing in the next 30 minutes, here's what we're waiting on").
- Separates **facts from hypotheses** in every update.
- Pulls in **legal and privacy** early when data or people are involved.
- **Manages fatigue.** Long incidents need handoffs. Exhausted responders make bad decisions.

## Containment: the core judgment call

Containment means stopping the attacker from doing more damage. It's where IR's hardest trade-offs live.

**Containment actions** might include: revoking sessions and tokens, disabling accounts, isolating hosts from the network, blocking IPs or domains, rotating credentials and keys, removing OAuth grants, pausing CI pipelines, tightening access policies.

**The central dilemma: act now, or watch?**

- **Act now:** limits immediate damage. But if your scoping is incomplete, the attacker notices, abandons the footholds you know about, and keeps the ones you don't. They may also escalate, destroy evidence, or speed up exfiltration.
- **Watch first:** lets you learn the full scope (all footholds, all compromised identities, the C2 channel) so you can evict them in one coordinated move. But the attacker keeps operating meanwhile, and you're accepting ongoing risk.

How to decide:

- **What's the attacker doing right now, and how close are they to the crown jewels?** If they're about to reach something critical, or actively exfiltrating, contain now.
- **How confident are you in your scope?** If you're fairly sure you know all footholds, coordinated eviction is achievable.
- **What's the adversary's likely sophistication?** A commodity attacker who gets kicked out often moves on. A state actor who gets kicked out often comes back through a door you haven't found.
- **What do partial actions reveal?** Revoking one session tells the attacker you've noticed. Sometimes quieter moves (adding monitoring, restricting the target asset's access) buy time without tipping them off.

There's no universal answer. What's expected of a D&R engineer is to **make the trade-off explicit**: "If we revoke now, we risk X. If we wait two hours to scope, we risk Y. I recommend Z because..." That's the skill interviewers probe with scenario questions.

**Containment also has business cost.** Isolating a production host may cause an outage. Disabling a founder's account mid-fundraise is disruptive. Rotating a widely used credential breaks things. Good responders know the systems well enough to estimate these costs, or know who to ask fast.

## Eradication and recovery

**Eradication** is removing every attacker foothold. It's where under-scoping (Chapter 5) bites. The checklist mindset helps: for each compromised identity and system, what persistence could the attacker have set up?

- Identity: added MFA devices, app passwords, OAuth grants, API tokens, changed recovery info, new federated trust.
- Cloud: new users, roles, access keys, modified trust policies, new resources (compute that could serve as a beachhead), modified logging configuration.
- Hosts: new SSH keys, scheduled jobs, services, modified startup items, implants.
- Code/CI: modified workflows, new deploy keys, poisoned dependencies, altered build scripts.

For high-severity compromise, the safest approach is often **rebuild rather than clean**: reimage the laptop, redeploy the infrastructure from known-good code, rotate every secret the compromised system could reach. Trying to surgically "clean" a compromised system relies on your having found everything.

**Recovery** is returning to normal operation with confidence. That includes monitoring closely for re-entry (attackers often try again via the same or adjacent route), and only gradually relaxing emergency restrictions.

## Communication

Communication failures sink incidents as often as technical failures. Some principles:

- **Use a dedicated channel**, and assume the attacker may be reading your normal ones. If the attacker might have access to corporate chat or email (it happens — chat and email are prime attacker targets), coordinate out of band.
- **Need-to-know.** Especially for insider cases, keep the circle tight.
- **Updates on a cadence** for leadership, even when nothing has changed ("no change since last update; next update at 16:00"). Silence breeds anxiety and interference.
- **Precise language.** "Compromised" vs "accessed" vs "exposed" vs "exfiltrated" mean different things legally and to customers. Don't say "breach" casually; it can have legal meaning.
- **Bring in legal early** for anything involving personal data, customer data, or potential regulatory notification. Notification deadlines (e.g. under GDPR, SEC rules for public companies, contractual obligations) can start from the moment of awareness.

## Postmortems

After an incident, a **blameless postmortem** (also called post-incident review) documents what happened and what to change. Key elements:

- **Timeline** of the attack and of the response.
- **Root cause and contributing factors**, which are usually several. "The engineer clicked a phishing link" is never a root cause. Why did one click lead to compromise? Why wasn't it detected sooner? Why could the attacker reach so much?
- **What went well**, so it's kept.
- **Detection analysis:** at which stages could we have detected this? Which detections fired, which should have and didn't, why? This section is the main input to detection engineering.
- **Action items** with owners and deadlines, prioritized. The test of a postmortem is whether its action items get done.

"Blameless" doesn't mean "no accountability." It means you focus on systems and incentives instead of individuals, because people hide mistakes when they fear blame, and hidden mistakes are how the next incident happens.

## Preparation, revisited

Most of what makes an incident go well happens before it starts:

- **Playbooks** for common scenarios (compromised identity, compromised laptop, leaked secret, malicious insider, compromised third-party vendor). Short, practical, kept current. Not a 40-page document nobody reads.
- **Access ready in advance.** Responders need the ability to revoke sessions, isolate hosts, pull logs, and query everything without waiting on access requests during an incident. Usually via a well-audited break-glass mechanism.
- **Relationships.** Knowing who owns which system and having their phone numbers. Having met legal before the crisis.
- **Exercises.** Tabletop exercises (talking through a scenario) and live simulations. The first time you run an "our IdP admin is compromised" scenario should not be the real one.
- **Pre-approved actions.** Agree in advance which containment actions D&R can take without asking (revoke a user's sessions, isolate a laptop) so there's no hesitation at 3am.

## Questions to think about

1. In your abuse work, have you ever held off on actioning accounts to learn more about a network first? What made that the right or wrong choice? How would the calculus change against a state actor?
2. You're IC. An attacker has an SSO session for an engineer and has been active for 20 minutes. Lay out what you'd do in the first 15 minutes, and what you'd *not* do yet.
3. Write in your head the "root cause" section for an account takeover incident you know, without blaming the user. What systemic factors would you list?
