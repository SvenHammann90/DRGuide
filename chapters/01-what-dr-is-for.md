# Chapter 1 — What D&R is for

Before going into how detection works, it helps to be precise about what a D&R team is *for*. Teams that aren't clear on this end up as alert factories: busy, exhausted, and not much safer.

## The mission in one line

> **Reduce the time and damage between an adversary gaining a foothold and that adversary being fully evicted.**

Everything a D&R team does should map to that. A few terms make it concrete:

- **Dwell time:** how long an attacker is inside before being detected. Industry-wide medians have come down over the years, from many months to days or weeks depending on the report, but serious, targeted intrusions still often go unnoticed for a long time.
- **Time to detect (TTD)** and **time to respond/contain (TTR)**: the two halves of dwell time you can actually influence.
- **Blast radius:** how much an attacker can reach from a given foothold. D&R doesn't own blast radius (that's architecture and access control), but it lives with the consequences and should push hard to reduce it.

Why time matters so much: an intrusion is not a single event. It's a **chain** of steps (get in, get established, look around, get more access, reach the target, take what they came for). Every step leaves traces and takes time. If you catch the attacker at step two, the incident is a laptop reimage and a password reset. If you catch them at step six, it's a breach disclosure.

That leads to one of the most useful mental models in D&R:

> **Defenders get many chances to catch an attacker. The attacker has to avoid detection at every step.**

This is the "defender's advantage" view, and it's the counterweight to the cliché that "the attacker only has to be right once." Inside your environment the attacker is on your turf, and every action risks tripping something. Good D&R programs put detection at many points along the chain, not just at the front door.

## The lifecycle

Most frameworks (NIST SP 800-61 is the classic reference) describe some version of this loop:

1. **Preparation:** telemetry, detections, playbooks, access, relationships, practice.
2. **Detection & analysis:** something fires or someone notices. Triage it, investigate, decide whether it's an incident.
3. **Containment:** stop it from getting worse.
4. **Eradication:** remove the attacker's access and persistence.
5. **Recovery:** restore systems and confidence.
6. **Post-incident learning:** figure out what to change so it doesn't happen again, or gets caught sooner.

Two things about this loop are easy to miss.

**First, most of the value is in phases 1 and 6.** Incidents are the visible part, but preparation decides how well an incident goes, and post-incident learning decides whether the next one goes better. Strong D&R engineers spend most of their time *not* responding to incidents. They build telemetry, write and tune detections, automate triage, run exercises, and feed lessons back into engineering.

**Second, the loop feeds prevention.** Every incident and every near miss is evidence about where prevention is weak. A good D&R team tells the rest of security, and the rest of engineering, "here's where attackers actually get in and move, so harden *this*." That influence is one of the main ways a D&R engineer creates value beyond their own team.

## Where D&R sits in a security organization

Security organizations vary, but you'll typically see something like:

- **Security engineering / infrastructure security:** builds secure systems: identity, access control, network architecture, secrets management, hardening.
- **Product / application security:** secure design reviews, code review, vulnerability management for the company's own software.
- **Corporate / IT security:** endpoints, SaaS, employee identity.
- **Detection & Response:** the subject of this guide. Sometimes split into detection engineering, incident response, and threat intelligence, sometimes one blended team.
- **Threat intelligence:** what adversaries are doing in the world and against us specifically. Sometimes inside D&R.
- **Red team / offensive security:** emulates attackers to test defenses.
- **GRC (governance, risk, compliance):** policies, audits, regulatory requirements.

A "full-stack" D&R engineer is one who can work across the whole D&R loop (get the telemetry in, write the detection, triage the alert, run the incident, automate the response, write the postmortem) rather than specializing in one slice. At smaller or fast-growing organizations, that breadth is the norm. It also means you're expected to *build*: D&R engineering roles are usually engineering roles, writing code, data pipelines, and infrastructure, not only operating a vendor console.

## The SOC model vs the engineering model

There are two broad philosophies for how D&R work gets done, and knowing the difference will help you read job descriptions.

**The traditional SOC model.** Tiered analysts (L1/L2/L3) watch a queue of alerts from a SIEM (Security Information and Event Management system). L1 triages, escalates to L2, and so on. Detection content often comes from vendors. Success is measured in alerts closed and SLAs met. This model scales headcount linearly with alert volume and tends to burn people out.

**The detection-engineering model** (common in tech companies). Detections are code: version-controlled, reviewed, tested, deployed through a pipeline. Engineers own their detections end to end, including the pain of triaging them, which gives them a strong incentive to make detections precise. Repetitive triage gets automated. Humans spend their time on judgment, not on clicking through the same benign alert for the 400th time. Success is measured in coverage, signal quality, and response outcomes.

Anthropic-style roles are firmly in the second camp. And the second model is where AI changes the most: when triage and enrichment can be heavily automated, the engineer's job moves further toward *deciding what to detect, judging whether the system is working, and handling the cases that need human judgment*. That matches the "set direction, judge outputs" framing you started with.

## How D&R knows whether it's doing well

This is a hard question, and you should have an opinion on it, because metrics drive behavior.

Bad or misleading metrics:

- **Number of alerts / detections.** More is not better. A thousand noisy detections is worse than a hundred good ones.
- **Alerts closed per analyst.** Rewards speed over correctness, and rewards *not* digging.
- **"Number of incidents."** Ambiguous. More incidents could mean worse security or better detection.

More useful indicators:

- **Coverage against relevant attacker behavior.** For the techniques that matter in our threat model, do we have telemetry, a detection, and a tested response? (Chapter 4 and 7.)
- **Signal quality.** True-positive rate per detection, time analysts spend on false positives, how many alerts lead to action.
- **Time to detect and contain** in real incidents and in exercises (red team, purple team).
- **Detection validation results.** When we simulate an attack technique, does the right alert fire, does it reach the right person, and do they know what to do?
- **How incidents were found.** If most real incidents are found by luck, by external notification, or by an employee noticing something odd rather than by your detections, that's a strong signal about coverage gaps.
- **Recurrence.** Are we seeing the same root causes again?

Notice the parallel with your world: in abuse, "accounts actioned" is the vanity metric and "abuse that got through, measured by sampling and audit" is the honest one. Same instinct here.

## Questions to think about

1. In your current work, what is the equivalent of "dwell time"? How long does a compromised account typically stay compromised before you notice, and what drives that number?
2. Why might a D&R team that handles very few incidents be either excellent or terrible? What would you look at to tell which?
3. If you could only measure one thing about a D&R team, what would you choose, and how could it be gamed?
