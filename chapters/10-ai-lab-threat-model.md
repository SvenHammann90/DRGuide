# Chapter 10 — The AI lab threat model

Everything so far applies to D&R at any tech company. This chapter is about what's distinctive at a frontier AI lab, and specifically about how to reason about it. It draws only on public information and first-principles thinking. It doesn't describe how Anthropic's security team actually operates, and you should treat it as a way of thinking, not a set of facts.

## Why AI labs are unusual targets

Most companies hold things valuable to criminals: customer data, money, access to their customers. AI labs hold those too, but they also hold assets of **strategic, national-level value**:

- **Model weights.** The trained parameters of a frontier model represent enormous compute investment, and whoever holds them can run, fine-tune, or strip safety training from the model. Stolen weights can't be "un-stolen."
- **Algorithmic insights and research.** Training techniques, data recipes, architecture choices, and unreleased results. Often small in bytes, large in value, and easy to exfiltrate (a document, a conversation, a config file).
- **Training and inference infrastructure:** large compute clusters that are valuable in themselves and that an attacker could use to tamper with training.
- **Safety systems:** the classifiers, monitoring, and policies that prevent misuse. An attacker who understands or undermines them can misuse models more effectively.

Because of that, frontier AI labs are widely understood to be targets of **state-level adversaries**, not just criminals. Public policy discussion (for example RAND's 2024 report *Securing AI Model Weights*, which defines security levels against progressively more capable attackers) treats protecting frontier weights against nation-state operations as a serious, unsolved problem.

## How Anthropic frames it publicly

Anthropic's **Responsible Scaling Policy (RSP)** ties security requirements to model capability: as models cross capability thresholds, the required security standard rises. Anthropic has publicly described activating its ASL-3 protections, which include a security standard aimed at protecting model weights from sophisticated attackers, with higher levels envisioned for more capable models and more capable adversaries. Public descriptions of these measures have included things like strict access controls around weights, multi-party authorization for sensitive access, and controls on data egress.

What that tells you about a D&R role there, as reasoning rather than inside knowledge:

- **Security is tied to the company's core mission and commitments**, not just compliance. D&R effectiveness against advanced adversaries is part of what lets the company keep its public commitments.
- **The bar is set by capable adversaries**, so the "assume breach" mindset and depth of detection (Chapter 2) matter more than usual.
- **Weights and research exfiltration are top scenarios.** Expect heavy emphasis on data-access telemetry, egress monitoring, and tightly controlled access paths to the most sensitive assets.
- **Insider risk is first-class.** When the asset is that valuable, coercion and recruitment of insiders is a realistic threat model, which is why multi-party controls exist. D&R would work alongside these controls, watching for attempts to circumvent them.

## Scenarios worth thinking through

If you were the D&R engineer, these are the kinds of "assume breach" questions you'd want answers to. Reasoning through them is great interview preparation.

**Weights exfiltration.** Where do weights live? Which identities and systems can read them? What does normal access look like (training jobs, inference serving, evaluation)? How would an attacker with a foothold on a researcher's laptop try to reach them? What would large or unusual data movement from those systems look like, and is egress constrained so that exfiltration is slow and noisy?

**Research theft.** Much valuable research isn't in big files but in docs, code, chat threads, and notebooks. How would you notice someone systematically collecting research material? This is a hard problem: low volume, legitimate-looking access, and an engineering culture where people read widely.

**Compromised researcher identity.** An attacker phishes or steals a session from a researcher with access to training infrastructure. What can they reach? What would distinguish them from the real researcher? (Chapters 5 and 8.)

**Training pipeline tampering.** An attacker who can modify training data, code, or configuration could try to plant backdoors in a model, a threat specific to AI. Detecting this sits at the intersection of supply-chain security and ML integrity.

**Abuse of internal model access.** Employees and internal tools often have privileged access to models (e.g. models without production safeguards, for research). Monitoring that access for misuse is a distinctive AI-lab problem.

**AI agents as identities.** Labs heavily use AI agents internally, including agents that write code, run jobs, and access systems. These are a new class of non-human identity with broad permissions and behavior that's less predictable than a classic service account, and which can be manipulated through prompt injection. How do you baseline them? How do you tell a compromised or misdirected agent from one doing its job? This is a frontier problem in D&R generally, and an AI lab is where it shows up first.

**Third parties and cloud providers.** Compute often runs at cloud providers and specialized partners. Visibility and control are shared with them. Incident response across that boundary needs planning in advance.

## Where D&R and Safeguards meet

Chapter 0 distinguished Security D&R (attacks against the company) from Safeguards / platform abuse (misuse of the product). At an AI lab they intersect more than elsewhere:

- **Attackers use the product.** Threat actors use AI models for reconnaissance, malware development, and operations. Signals of that misuse, which are visible on the product side, can be threat intelligence about attacks against the company or its customers.
- **Customer account compromise can lead into infrastructure.** API keys, workspace admin accounts, and integrations are both product-abuse and security concerns.
- **Shared methods.** Both use detection, investigation, and response; both increasingly use AI to do it.

This intersection is a real asset for your story. You can credibly say: "I understand how adversaries abuse the platform side, and I want to defend the company side; at an AI lab, those connect."

But be careful not to pitch yourself *only* as a platform abuse person. The role is about defending the organization's infrastructure and identities. Show that you understand and are drawn to that.

## Culture and constraints you should expect

Reasonable expectations for security at a research-heavy, fast-growing AI lab, again as inference:

- **Researchers and engineers need freedom to move fast.** Security that blocks research gets routed around. D&R has to be precise enough not to drown people in false alarms, and to make strong controls low-friction.
- **Rapid growth** means the environment changes constantly: new systems, new people, new compute. Detections and context data go stale quickly.
- **Heavy, early use of AI** in the security function itself. Expect to be asked how you'd use AI to scale detection and response, and how you'd make sure it's trustworthy (Chapter 9).
- **Mission alignment.** Security there is part of making advanced AI go well. Being able to say why that matters to you is relevant, and not just a formality.

## Questions to think about

1. Sketch the attack graph from "attacker has a phished session for a research engineer" to "attacker has a copy of model weights." Where are the best detection points? Where are the best prevention points?
2. Why might research theft be harder to detect than weights theft? What signals would you even look for?
3. An internal AI agent with broad repo access suddenly starts reading files in areas it has never touched. List hypotheses (benign and malicious) and how you'd discriminate between them.
