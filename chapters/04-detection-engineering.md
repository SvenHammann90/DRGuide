# Chapter 4 — Detection engineering

Detection engineering is the craft of turning "attackers might do X" into a reliable signal that reaches the right human (or automation) with enough context to act. It's the core skill of a D&R engineer, and it's where your anti-abuse instincts about signal quality are most useful, and also where they most need adjusting.

## The base rate problem, again

Start with the arithmetic, because it shapes everything.

Suppose you have a detection that is 99.9% specific (only 0.1% of benign events trigger it) and it runs on 10 million events a day. That's **10,000 false positives a day**. Even if it catches every real attack, a real attack might be one event in a month. Your analysts will drown, start ignoring the alert, and miss the real one when it comes.

In abuse, bad traffic is often 1–30% of a stream, so a 99.9%-specific signal is excellent. In D&R, bad activity is often one in a billion, so "pretty specific" is useless. **Detections must be extraordinarily precise, or they must run on a much smaller, pre-filtered set of events.**

This is why D&R detections tend to:

- Target **rare, specific behaviors** (an admin role granted to a new identity; a shell spawned by a web server process; a login to the cloud root account) rather than general suspicion.
- **Narrow the population** before applying logic: "among production hosts" or "among identities with access to model weights."
- **Layer conditions:** rare event A *and* context B *and* no explanation C.

## Alert fatigue is a security failure, not an inconvenience

Every famous breach review seems to include the line "the alert fired, but it was buried among thousands of others." Noisy detections don't just waste time. They train people to dismiss alerts, and they hide the real one.

So a detection engineer owns not just "does it catch the attack?" but "**what is the total cost of this detection to the people who triage it?**" A good team tracks, per detection: volume, true-positive rate, average triage time, and how often it leads to an action. Detections that never produce action get tuned or retired.

A useful principle from many mature teams: **an alert should mean someone needs to do something.** If the right response to an alert is usually "look at it and close it," then it's not an alert, it's a report, and it belongs in a dashboard or a hunt, not a page.

## Kinds of detection logic

Roughly from simplest to most complex:

**Atomic / signature detections.** Match a single known-bad thing: a known malicious hash, a known-bad domain, a specific command line. High precision when it hits, brittle against change. Cheap to have, and useful for known campaigns.

**Behavioral detections.** Match a pattern of behavior that's suspicious regardless of the specific tool: "a process reads browser cookie databases and isn't the browser itself," "a new IAM user is created and immediately given admin permissions," "an OAuth app is granted broad mailbox scopes." These are the backbone of good detection programs, because they sit higher on the Pyramid of Pain.

**Threshold / aggregation detections.** Count something over time: "more than N failed MFA pushes for one user in 10 minutes" (MFA fatigue), "an identity downloads more than X GB from the code host in an hour." Familiar territory from abuse rate-limiting.

**Sequence / correlation detections.** Link events across time or sources: "a password reset via help desk, followed by a new MFA device enrollment, followed by a login from a new country, within two hours." These are powerful against identity attacks, and depend heavily on joinable telemetry (Chapter 3).

**Anomaly detections.** Compare behavior to a baseline: "this user has never accessed this system before," "this service account is calling an API it has never called." Here's the honest take:

- Anomaly detection in D&R is valuable mostly **as context and as a filter**, not as a standalone alert. "First time this user accessed the weights bucket" is a great enrichment for an alert, and a decent hunting lead, but alone it fires constantly in a growing company where people change projects every month.
- **Unusual is not malicious.** Engineers are unusual all the time. A capable attacker who uses valid credentials during business hours from a VPN can look entirely normal.
- Where anomaly approaches shine: **narrow, stable populations** (service accounts, production infrastructure, privileged admins) where behavior really is predictable, and peer-group comparisons ("nobody else in this team accesses this system").

**Risk-based alerting.** Instead of every rule paging someone, rules contribute **risk points** to an entity (a user, a host). When an entity accumulates enough risk across different kinds of suspicious activity in a time window, a single higher-confidence alert fires. This is a clean way to use many weak signals without drowning in them, and it will feel very familiar: it's structurally a risk score, like account risk scoring in anti-abuse.

## Detection as code

In the engineering model, a detection is a piece of software:

- **Written in a query language or code** (SQL, a SIEM's language, Python, or a portable format like **Sigma**).
- **Version-controlled and peer-reviewed,** with a description of what it detects and why.
- **Tested,** both with unit-test-style sample events (true positives and known benign cases) and, ideally, with real attack simulations (Chapter 7).
- **Deployed through a pipeline,** with the ability to roll back.
- **Monitored** for volume, errors, and silent failure (a detection that throws an error and never runs is the worst kind of blind spot).

A good framework to know by name is **Palantir's Alerting and Detection Strategy (ADS) framework**. It's a template every detection should be documented against, roughly:

1. **Goal:** what behavior this detects.
2. **Categorization:** which ATT&CK tactic/technique.
3. **Strategy abstract:** how it works, in plain language.
4. **Technical context:** what the analyst needs to understand.
5. **Blind spots and assumptions:** what it will miss and what it relies on. (This is the most important section, and the most often skipped.)
6. **False positives:** known benign causes.
7. **Validation:** how to produce a true positive to test it.
8. **Priority:** how urgent an alert is.
9. **Response:** what to do when it fires.

Even if you never fill out a formal ADS, these nine questions are an excellent checklist for judging a detection, including one an AI wrote for you.

## How to judge a detection

When you review a detection (yours, a colleague's, or an AI's), ask:

1. **What attacker behavior is it actually about?** If you can't say in one sentence, the detection is unfocused.
2. **Is it on something the attacker must do, or something they happened to do?** (Chapter 2.) Hardcoded tool names and IPs are fragile. Behaviors are durable.
3. **How would an attacker evade it, and how hard is that?** Rename the binary? Use a different API that does the same thing? Go slower than the threshold? A detection that's trivially evaded is fine as a tripwire for the unsophisticated, but don't count it as coverage against capable adversaries.
4. **What's the expected volume and precision?** Has anyone run it over the last 30–90 days of data to see what it would have fired on? (This "backtest" is the single most useful thing to do before shipping.)
5. **What are the known benign causes, and are they handled by context rather than blanket exclusions?** Beware the exclusion that quietly creates a hole: "ignore everything from the platform team's service account" is exactly where an attacker would want to be.
6. **Does the alert carry enough context to act on?** Who is the user, what's normal for them, what happened before and after?
7. **What's the response?** If it fires at 3am, what does the on-call person do first?
8. **Does it depend on telemetry that might silently break?** How would we know?

## Precision and recall, D&R flavor

You know the precision–recall trade-off well. A few D&R-specific twists:

- **Recall is mostly unmeasurable directly.** You don't know how many intrusions you missed. You estimate recall through **adversary simulation** (red team, purple team, automated attack emulation) rather than labeled data.
- **Different detections deserve different trade-offs.** For crown-jewel access paths, accept more false positives and invest in fast triage. For noisy, low-impact areas, demand high precision.
- **Detection in depth beats one perfect rule.** Because an attacker must pass many stages, several medium-recall detections at different stages can give high recall for the intrusion as a whole. You don't need to catch them at every step, just at *some* step, early enough.

## The detection lifecycle

Detections rot. Environments change (new SaaS tools, new cloud services, reorganized teams), log formats change, attackers change. A healthy program treats detections as a living portfolio:

1. **Ideation:** from threat intel, incidents, red team findings, new systems, hunts.
2. **Research:** what does this look like in our telemetry? Is the telemetry even there?
3. **Build & test:** write it, backtest it, test with simulated true positives.
4. **Deploy:** with a staging period (log-only, not paging) if it's uncertain.
5. **Tune:** watch volume and precision, refine.
6. **Validate periodically:** does it still fire when the behavior happens?
7. **Retire:** when it's superseded, no longer relevant, or not worth its cost.

## Where ML fits, honestly

Coming from anti-abuse you might expect more ML in D&R. Where it does work well:

- **Specific, well-labeled problems** with lots of data: phishing email classification, malicious domain or URL classification, malware classification (mostly inside vendor products).
- **Prioritization and enrichment:** ranking alerts by likely severity, clustering similar alerts.
- **Baselining narrow populations,** as described above.
- **LLM-based triage and investigation assistance** (Chapter 9), which is the biggest current shift.

Where it tends to disappoint: general-purpose "detect any intrusion" anomaly models across a whole environment. They produce noise that's hard to explain and can't easily be tuned. Plenty of security products marketed as "AI-powered" fail exactly here. Your ML literacy is an asset in being skeptical of these claims.

## Questions to think about

1. Take a signal you use in anti-abuse. Rewrite it as a D&R-style detection: what's the behavior, what's the population, what's the expected volume, what are the benign causes, what's the response?
2. Why is "exclude service account X from this detection" one of the most dangerous changes a detection engineer can make? When is it still the right call?
3. An AI assistant hands you a detection for "suspicious cloud credential use." Using the eight review questions, what would you check first?
