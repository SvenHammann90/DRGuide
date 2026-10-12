# Chapter 5 — Triage and investigation

An alert is a question, not an answer. Triage decides whether the question is worth answering now. Investigation answers it. This chapter is about the reasoning behind both, which matters more than any query language.

## Triage: the first five minutes

Triage is a fast, bounded decision: **is this benign, malicious, or unknown, and how urgent is it?** The goal is not to fully understand the alert. It's to route it correctly.

A good triage pass usually checks:

1. **What fired and why?** Read the detection's description. What behavior was it looking for?
2. **Who and what is involved?** The user, host, service account, resource. What's their role, their normal behavior, their access?
3. **What's the potential impact if this is real?** An alert on an intern's test account is different from the same alert on an identity with production admin.
4. **Is there an obvious benign explanation, and can I *verify* it?** "That's probably the deploy script" is a hypothesis. "Here's the deploy script's run log with the same timestamp" is verification.
5. **What happened just before and just after?** A suspicious event surrounded by normal activity from the same session is different from one preceded by a login from a new country.

Then one of three outcomes:

- **Close as benign,** with a note on *why* (so the detection can be tuned and so the decision can be audited later).
- **Escalate to investigation.**
- **Declare an incident** if the evidence already suggests real compromise.

The most important triage discipline: **"I can't explain it" is not the same as "it's benign."** Under time pressure the temptation is to close things that look *probably* fine. Unexplained alerts on high-value assets should be escalated, not closed.

## Investigation as hypothesis testing

Investigation is the scientific method under time pressure. The trap that catches novices (and AI assistants) is **forming a story early and then collecting evidence that fits it.** Good investigators do the opposite.

A practical loop:

1. **State the competing hypotheses.** Not just "is this an attacker?" but: "this is the legitimate user traveling," "this is the legitimate user on a new VPN," "this is an attacker with a stolen session," "this is an automated job running under the user's identity."
2. **For each hypothesis, ask: what would I expect to see if it were true, that I wouldn't see if it were false?** This is the key move. Look for **discriminating evidence**, not merely consistent evidence.
   - Legitimate travel: the user's other devices and accounts show the same location; their calendar or chat shows travel; the login used their enrolled hardware key.
   - Stolen session cookie: the session was created in one place and suddenly used from another without re-authentication; the user agent or device fingerprint changes mid-session; the real user is simultaneously active from their usual location.
3. **Go get that evidence,** starting with what's fastest and most decisive.
4. **Update, and repeat.**

If you've done account takeover investigations you've done exactly this, just inside one product. D&R stretches it across many systems.

## Pivoting

**Pivoting** means using one piece of evidence to find related evidence: take the IP address from a suspicious login and search every log source for it; take the session ID and see everything it did; take the process hash and look for it on every host; take the OAuth app ID and see who else authorized it.

Pivoting is how an investigation grows from one alert to a full picture. It's also where it can sprawl. Discipline:

- **Pivot with a question in mind.** "Did this IP touch anything else?" is focused. "Let me look at everything" is not.
- **Note what you pivoted on and what you found,** including dead ends. Negative results matter ("this IP appears nowhere else in 90 days of logs").
- **Watch for shared infrastructure.** A VPN exit node, a corporate proxy, or a cloud provider IP range will connect unrelated activity. Your abuse instincts about shared IPs apply directly.

## Build a timeline

The central artifact of any investigation is the **timeline**: an ordered list of what happened, with timestamps, sources, and confidence. It's how you:

- Answer "when did this start?" (the first malicious event, which is often earlier than the event that triggered the alert).
- Spot gaps: periods where you can't see what the actor did.
- Communicate to others in an incident.
- Find the **initial access vector**, without which you can't be sure the attacker won't simply come back the same way.

Watch for timestamp pitfalls: different sources in different time zones, clock skew between systems, the difference between "when it happened" and "when it was logged."

## Scoping: the question that matters most

Once you know something malicious happened, the crucial question is **scope**: what did the attacker touch, and what could they have touched?

- **Which identities** were compromised? (Including ones created or modified by the attacker.)
- **Which hosts?**
- **Which data** was accessed or exfiltrated?
- **Which persistence mechanisms** did they set up? (New keys, tokens, OAuth grants, MFA devices, SSH keys, scheduled jobs, modified CI pipelines.)
- **What credentials could they have harvested** from where they were? Assume anything readable from a compromised system was taken: tokens in config files, cookies in browsers, secrets in environment variables.

The asymmetry here is important. **Under-scoping is much more dangerous than over-scoping.** If you evict the attacker from three of four footholds, they're still in. A common failure is to fix the obvious compromised account, declare victory, and miss the cloud access key the attacker minted on day one.

## Evidence and confidence

Be precise about what you know versus infer. A useful habit is labeling statements:

- **Observed:** "CloudTrail shows `GetObject` calls on bucket X from access key Y at 14:02."
- **Inferred:** "Access key Y was likely created by the attacker, because it was created from the same session that logged in from the unfamiliar IP."
- **Assumed / unknown:** "We don't log data events on bucket Z, so we can't say whether it was accessed."

The phrase "**absence of evidence is not evidence of absence**" matters most when the logs don't exist. "We found no exfiltration" is a very different claim from "we have complete egress logs and they show no exfiltration." In incident reports and in leadership updates, be explicit about which you mean.

Also preserve evidence before you change things. Containment actions (reimaging a laptop, deleting a malicious key) can destroy the evidence you need. Capture what you need first when you can. Formal forensic chain-of-custody matters mostly if legal action is likely, but basic evidence preservation matters always.

## Cognitive traps

Investigation failures are mostly reasoning failures, and they apply equally to you and to an AI assisting you:

- **Confirmation bias:** collecting only evidence that fits the first hypothesis.
- **Anchoring:** fixating on the first interpretation of the alert.
- **Normalizing:** "this user is always weird" or "that system always throws these," which is exactly where an attacker would hide.
- **Premature closure:** stopping once you have *a* plausible explanation rather than *the* explanation.
- **Authority bias:** "the user said it was them" (attackers may control the channel you're asking on; verify identity out of band).
- **Sunk cost:** continuing down a dead-end line because you've already spent an hour on it.

A good antidote is to ask, at each decision point: **"What would I expect to see if I were wrong?"** and then go check.

## Talking to the user

A distinctive part of corporate D&R: you can often just *ask*. "Hi, we saw a login to your account from Lisbon at 3am. Was that you?" This is fast and often decisive, but:

- Ask through a **verified channel** (not the possibly-compromised one).
- Be careful if you suspect **insider activity**. Asking may tip them off. These cases usually involve HR and legal before any contact.
- Remember users misremember. "Yes that was me" sometimes means "I didn't really read your message."

## Questions to think about

1. Recall an account takeover case where you were wrong at first. What evidence would have discriminated between your hypotheses earlier?
2. An alert shows an engineer downloading 40 GB from an internal data store at 11pm. List four hypotheses and, for each, one piece of evidence that would distinguish it from the others.
3. Why is "we found no evidence of data exfiltration" a dangerous phrase to put in a report? How would you phrase it more honestly?
