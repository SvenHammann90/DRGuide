# Chapter 1 — Organizations, accounts, and blast radius

Before getting into individual resources, it helps to understand how a company's cloud estate is organized at the top level. The structure isn't just bookkeeping. It's one of the strongest security controls a company has, and one D&R relies on constantly.

## The account (or project) as a wall

Every cloud resource lives inside a container: an **account** in AWS, a **project** in GCP, a **subscription** in Azure. This guide will say "account" for all three.

An account is a strong isolation boundary. By default, nothing in one account can see or touch anything in another. A credential that's valid in the "staging" account has no power in the "production" account unless someone has deliberately connected them.

That makes accounts the main tool for limiting **blast radius**. A mature company doesn't run everything in one account. It runs dozens or hundreds, separated by purpose:

- Production vs staging vs development.
- One account per major service or team.
- Separate accounts for security tooling and log storage.
- Separate accounts for especially sensitive assets (for an AI lab, model weights and training infrastructure are obvious candidates).
- "Sandbox" accounts where engineers can experiment freely, isolated from anything real.

When an attacker compromises a credential, one of the first scoping questions is: **which account is it valid in, and what's connected to that account?** A leaked key for a sandbox account is a minor incident. A leaked key for the production account that holds customer data is a major one.

## The organization: rules above the accounts

Accounts are grouped into an **organization** (AWS Organizations, a GCP organization, an Azure tenant), usually arranged in a tree: the organization at the top, then groups (AWS calls them organizational units, GCP calls them folders, Azure calls them management groups), then accounts at the leaves.

The organization lets security teams set **guardrails that apply to every account below a point in the tree**, no matter what permissions exist inside the accounts. Examples:

- AWS **Service Control Policies (SCPs)** and GCP **Organization Policies** can say things like "no one in these accounts may disable audit logging," "no resources may be created outside these regions," "no one may create long-lived service account keys," or "no bucket may be made public."
- Even an administrator *inside* an account can't override a guardrail set above it.

This is a powerful idea for defenders. Guardrails turn "we hope nobody does X" into "nobody can do X." And for D&R, they make certain detections unnecessary (if logging can't be disabled, you don't need to detect it being disabled, though you still should, because guardrails can be misconfigured or changed).

## The management account: the keys to everything

At the top of the tree sits a special account: the AWS **management account** (formerly "master account"), or the organization-level admin roles in GCP and Azure. Whoever controls it can change the guardrails, create and delete accounts, and often gain access to any account in the organization.

That makes it one of the most sensitive things in the whole company. Good practice:

- Run nothing in the management account except organization management.
- Very few people have access, with phishing-resistant MFA and strong logging.
- Any activity there is rare enough that it's worth looking at every time.

A related concept is the **root user**: in AWS, each account has an original identity with unlimited power over that account, created when the account was opened. Good practice is to lock it away (strong MFA, no access keys, used only for a handful of special tasks). **Any use of a root user is a high-priority detection**, because legitimate use is so rare.

## How accounts connect

Accounts are isolated by default, but real systems need them to work together. A service in one account reads data in another. A central security account reads logs from all the others. Engineers log in once and reach many accounts. These connections are made deliberately, usually through:

- **Cross-account roles:** a role in account B that trusts identities from account A to "assume" it (Chapter 2 explains assuming roles).
- **Resource policies:** a bucket in account B with a policy that allows a specific identity in account A to read it.
- **Central identity:** single sign-on (SSO) through the company's identity provider, which lets employees sign in once and choose which account and role to use.

For D&R, every connection is a potential path for an attacker. The questions to keep asking are:

- Which accounts trust which other accounts?
- Does any account trust something *outside* the organization? (A role that trusts an external account ID is a classic persistence technique, and also a classic legitimate vendor integration. Telling them apart takes context.)
- If an attacker holds a foothold in account A, what can they reach in other accounts?

## Landing zones and account factories

Mature companies create accounts through automation, often called an **account factory** or **landing zone**. Every new account starts with the same baseline: logging turned on and shipped to the central security account, guardrails applied, standard roles created, security tooling installed.

For D&R this matters in two ways:

1. **Consistency is coverage.** If every account is created the same way, you can trust that the audit log is on everywhere. Accounts created by hand, outside the factory, are where blind spots hide. A good detection is "an account exists that isn't sending logs to the central log archive."
2. **Deviation is signal.** If the baseline says "nobody creates IAM users in this account," any IAM user created there is either a mistake or an attacker.

## Regions

Cloud providers run in many geographic **regions**. Most resources live in one region. Two things matter for D&R:

- **Attackers like unused regions.** If your company only uses two regions, an attacker launching crypto-mining machines in a third region hopes nobody is looking there. Good guardrails block unused regions entirely. Good detections flag activity in unexpected regions.
- **Some logs are regional.** Make sure logging covers every region, including the ones you don't use, because those are exactly where an attacker might go.

## What this means for you as a defender

When you join a D&R team, one of the first things to learn is the shape of the organization:

- How many accounts exist, and how they're grouped.
- Which accounts hold the crown jewels.
- Which guardrails exist and what they block.
- Who can access the management account and root users.
- Which cross-account and external trust relationships exist.

This is your map. Without it, you can't tell whether an alert is about a sandbox nobody cares about or the account that holds model weights.

## Questions to think about

1. Why might an AI lab put its model weights in a dedicated account rather than alongside the services that use them? What would that protect against, and what wouldn't it?
2. A role in your production account trusts an AWS account ID you don't recognize. What hypotheses would you consider, and what evidence would distinguish them?
3. If you could set only three organization-wide guardrails, which would you choose, and why?
