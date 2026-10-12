# Chapter 7 — Proactive work: hunting, validation, purple teaming

Detections only catch what someone thought to write a detection for, and only if the telemetry and logic actually work. Proactive work exists to cover both gaps: **finding attackers your detections miss**, and **proving your detections do what you think.**

## Threat hunting

**Threat hunting** is a human-led search for malicious activity that hasn't triggered an alert. It starts from the assumption that an attacker might be present and undetected, and goes looking.

Good hunts are **hypothesis-driven**, not "let's look around":

- *Intel-driven:* "A threat actor targeting AI companies has been reported stealing browser session cookies and replaying them through residential proxies. Has any session in our IdP been used from a different ASN than it was created on, without re-authentication?"
- *Technique-driven:* "Attackers establishing persistence in cloud accounts often create access keys for existing users. Have any long-lived access keys been created in the last 90 days outside our provisioning automation?"
- *Environment-driven:* "We just onboarded a new SaaS tool with admin APIs. Who has used those APIs and for what?"
- *Incident-driven:* "In last month's incident the attacker used technique X. Is it present anywhere else?"

A hunt has three possible outcomes, all useful:

1. **Found something malicious:** start an incident.
2. **Found nothing malicious, but learned what normal looks like:** turn that into a detection (if the hypothesis can be expressed as a precise rule) or document the baseline.
3. **Couldn't answer the question:** you've found a telemetry gap. Fix it.

The third outcome is underrated. Many hunts end with "we can't actually tell," and that's a valuable finding.

The best output of hunting is usually **new or improved detections.** A hunt you repeat by hand every month should probably become automated.

## Detection validation

You wrote a detection. Does it work? Many don't, and the reasons are mundane:

- The log source stopped flowing, or the field name changed in a vendor update.
- The detection has a bug that means it never matches.
- An exclusion added during tuning accidentally excludes the real attack.
- The alert fires but routes to a channel nobody watches.
- The alert fires, the analyst sees it, and doesn't know what to do.

**Validation** means deliberately producing the behavior the detection is meant to catch, in a controlled way, and checking the whole chain: telemetry → detection → alert → triage → response.

Approaches:

- **Unit tests** on sample events (catches logic bugs, not telemetry problems).
- **Atomic tests:** run small, safe simulations of specific techniques (the open-source **Atomic Red Team** library is a common reference) on test systems and check the alert fires.
- **Continuous automated validation:** scheduled simulations that run regularly and alert if the expected detection *doesn't* fire. This catches silent breakage.
- **Canary / heartbeat checks** on log sources: alert when a source's volume drops to zero or changes sharply.

The mindset: **a detection you haven't seen fire on a true positive is a hypothesis, not coverage.**

## Red teaming and purple teaming

**Red teams** emulate real adversaries end to end, usually with limited knowledge shared with defenders, to test whether the organization can detect and respond to a realistic attack. Their output: what they achieved, how, and what (if anything) caught them. The honest red team report is often humbling and always valuable.

**Purple teaming** is collaborative: red and blue (defenders) work together, technique by technique. The red side executes a technique; the blue side checks what telemetry they see, whether a detection fired, and if not, builds or fixes one on the spot. Less realistic than a covert red team exercise, but far more efficient for improving coverage quickly.

What a D&R engineer should take from both:

- For each technique: **did we have telemetry? did a detection fire? was it triaged correctly? how fast?** That gives a much more honest coverage picture than an ATT&CK heat map built from detection metadata.
- **Patterns of failure** matter more than individual misses. "We consistently miss things that happen through legitimate admin tools" is a strategic finding.

## Tabletop exercises

A **tabletop** is a structured discussion of a scenario: "It's Tuesday 10am. Our cloud provider emails us that an access key belonging to our CI system was found in a public paste. Go." Participants walk through what they'd do, and the facilitator injects complications ("legal wants to know if customer data is involved," "the key's owner is on a plane").

Tabletops are cheap and very revealing. They tend to expose:

- Missing access ("who can actually revoke that key?").
- Missing telemetry ("do we even log what that key did?").
- Unclear ownership and decision rights.
- Communication gaps.

D&R engineers often design and run these, which is also good practice for the scenario-style interview questions you'll face.

## Feeding prevention

A theme from Chapter 1 worth repeating here: proactive D&R work produces a stream of findings that *aren't* detections:

- "These 40 service accounts have admin rights they don't use."
- "Long-lived cloud access keys are still being created by hand."
- "This SaaS tool lets any user install third-party apps with full data access."

The best D&R teams turn these into **hardening recommendations** and push them to the teams who own those systems. Reducing what attackers can do is often more valuable than one more detection on it. Knowing when to say "this should be prevented, not detected" is a sign of seniority.

A simple decision rule: **if the behavior is never legitimate and can be blocked without much cost, block it. If it's sometimes legitimate, or blocking is too costly, detect it.** Detecting something that could simply be prevented is a waste of attention.

## Questions to think about

1. In anti-abuse, what's the equivalent of a hunt: something you looked for proactively without an alert? Did it lead to a new automated signal?
2. Why might a team with "100% of its detections passing unit tests" still miss a real attack? List three reasons.
3. Pick a behavior you'd currently detect. Should it be prevented instead? What would make prevention too costly?
