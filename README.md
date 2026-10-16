# From Account Abuse to Detection & Response

A reading guide for someone who knows anti-abuse work (compromised accounts, fake and bulk-created accounts, abusive users) and wants to move into a full-stack Detection & Response (D&R) security engineering role, for example Security Engineer, Detection & Response at an AI lab like Anthropic.

It is meant to be read, not worked through. There are no labs or exercises. Each chapter is 15–25 minutes of reading and ends with a few questions to think about on the way home.

The focus is judgment: what matters in D&R, why, and how a good D&R engineer thinks. Tools and query languages change, and you will lean on AI for most of the syntax. What you can't hand off is knowing which question to ask, which answer to trust, and which risk to accept. That is what this guide is about.

## How to read it

Read the chapters in order the first time. Later chapters assume the vocabulary from earlier ones. After that, chapters 4, 6, and 11 are the ones worth rereading.

| # | Chapter | What you get out of it |
|---|---------|------------------------|
| 0 | [Orientation: why D&R feels different](chapters/00-orientation.md) | The core conceptual shift from anti-abuse to D&R |
| 1 | [What D&R is for](chapters/01-what-dr-is-for.md) | Mission, lifecycle, how the team fits into security |
| 2 | [Thinking like the adversary](chapters/02-adversary-thinking.md) | Threat models, attack chains, ATT&CK as a shared vocabulary |
| 3 | [Telemetry: you can only detect what you can see](chapters/03-telemetry.md) | Log sources, coverage, and the data engineering underneath D&R |
| 4 | [Detection engineering](chapters/04-detection-engineering.md) | How good detections are designed, tested, and maintained |
| 5 | [Triage and investigation](chapters/05-triage-and-investigation.md) | Getting from an alert to the truth |
| 6 | [Incident response](chapters/06-incident-response.md) | Running an incident: containment, decisions under uncertainty, communication |
| 7 | [Proactive work: hunting, validation, purple teaming](chapters/07-proactive-work.md) | Finding what your detections miss |
| 8 | [Modern attack surface: identity, cloud, supply chain, insiders](chapters/08-modern-attack-surface.md) | Where real intrusions happen today, and where your background helps most |
| 9 | [AI in D&R: directing and judging the machine](chapters/09-ai-in-dr.md) | Using LLMs and agents well, and where they fail |
| 10 | [The AI lab threat model](chapters/10-ai-lab-threat-model.md) | What is specific about defending a frontier AI company |
| 11 | [Bridging your background](chapters/11-bridging-your-background.md) | What transfers, what doesn't, and how to tell your story |
| 12 | [Glossary and further reading](chapters/12-glossary-and-reading.md) | Terms, plus a short list of material worth your time |

## Follow-up guide

[Inside the Cloud: A D&R Reader's Guide](cloud/README.md) goes one level deeper into the environment you'd be defending: cloud organization and accounts, IAM, keys and secrets, Kubernetes, storage, networking, pipelines, cloud logging, two worked incidents, and the AI lab cloud.

## One caveat

Everything in chapter 10 about Anthropic comes from public material and general reasoning about the threat model of a frontier AI lab. It does not describe Anthropic's internal setup. Treat it as a framework for thinking about the role, not as facts about the team.
