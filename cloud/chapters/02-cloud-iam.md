# Chapter 2 — Cloud identity and IAM

**IAM** (Identity and Access Management) decides who can do what in the cloud. Since everything is an API call, IAM is the real perimeter. It's also famously complex, and most serious cloud incidents involve IAM somewhere: a credential that shouldn't have existed, a permission that was too broad, or a trust relationship nobody understood.

You don't need to write IAM policies fluently. You do need a clear picture of the moving parts, because every investigation runs through them.

## Two questions: who are you, and are you allowed?

Every API call is checked twice:

1. **Authentication:** who is making this request? The request carries a credential (Chapter 3), and the cloud verifies it and attaches an identity.
2. **Authorization:** is that identity allowed to perform *this action* on *this resource* right now? The cloud evaluates every relevant policy and says yes or no.

Most of IAM complexity is in the second question.

## Principals: the "who"

A **principal** is anything that can make a request. There are two broad families, and the difference matters a lot.

**Human principals.** Employees and contractors. In a well-run company, humans don't have their own credentials inside the cloud. They sign in through the company's identity provider (Okta, Google Workspace, Entra ID) using SSO, and the cloud gives them temporary access to a role. This is called **federation**. It means that turning off someone's account in the identity provider turns off their cloud access too, and that MFA and device checks apply to cloud access.

Some companies still have **IAM users** with their own passwords and long-lived keys created directly in the cloud. These are a common source of trouble, since they live outside the identity provider's controls.

**Non-human principals (workload identities).** Code that needs to call the cloud API: a web service reading from a database, a training job writing checkpoints, a CI pipeline deploying infrastructure. In AWS these usually use **roles**. In GCP they use **service accounts**. In Azure they use **managed identities**.

Non-human identities usually outnumber humans by a wide margin, often by ten to one or more. They're often over-privileged ("just give it admin so it works"), rarely reviewed, and not protected by MFA. That makes them a favorite target.

## Roles and assuming them

A **role** is an identity with a set of permissions but no permanent credentials of its own. Someone or something has to **assume** it. When you assume a role, the cloud hands you short-lived credentials (typically valid for one to twelve hours) that act as that role.

Each role has two policies:

- A **trust policy:** who is allowed to assume this role? ("Engineers signing in through our SSO," "the CI system from this repository," "this other account.")
- A **permissions policy:** what can the role do once assumed?

Role assumption is the backbone of cloud access. An engineer signs in through SSO, assumes a "developer" role in the staging account, then later assumes an "admin" role in production if they're allowed. A service running on a VM automatically gets credentials for the role attached to that VM.

For D&R, role assumption creates a specific challenge: **the identity chain**. When you look at an audit log entry, you'll see "role X did this." But who was acting as role X? You often have to follow the chain backwards: role X was assumed by session Y, which came from SSO login Z, which was user Alice on her laptop. Providers record pieces of this chain (AWS has a "source identity" field and session names, for example), but connecting it end to end is real work and a recurring theme in cloud investigations.

## Policies: the "what"

A **policy** is a document that grants or denies permissions. Conceptually, each statement in a policy says:

> **Principal** may (or may not) perform **actions** on **resources**, under **conditions**.

For example, in plain language:

- "The training-job role may *read and write* objects in the *checkpoints bucket*."
- "Engineers may *start and stop* instances, but only if they *signed in with MFA*."
- "Nobody may *delete* the *audit log bucket*."

Things to know:

**Default deny.** If nothing grants a permission, it's denied. Access must be explicitly allowed.

**Explicit deny wins.** If any applicable policy says "deny," it doesn't matter how many say "allow." That's what makes guardrails (Chapter 1) work: a deny at the organization level overrides any allow inside the account.

**Policies come from several places.** Permissions attached to the identity, policies attached to the resource itself (a bucket policy saying who can read it), guardrails at the organization level, permission boundaries, session policies. The final answer combines all of them. This is why "can this identity do X?" is surprisingly hard to answer by reading, and why providers offer tools that simulate the evaluation.

**Wildcards are where danger hides.** A policy that allows `*` actions on `*` resources is full admin. Policies that allow `s3:*` or `iam:*` are close to it. When reviewing permissions, wildcards are the first thing to look for.

**Conditions add context.** Policies can require that a request comes from a specific network, a specific organization, with MFA, during a time window, or with specific tags. Conditions are how mature environments express "only from our corporate network" or "only resources inside our organization."

## Privilege escalation in IAM

Cloud IAM is flexible enough that some permissions let you grant yourself more permissions. Researchers have catalogued dozens of such paths. You don't need the list, just the pattern. Examples in plain terms:

- **Editing your own policy.** If you can modify the policy attached to your own role, you can make yourself admin.
- **Creating credentials for someone else.** If you can create access keys for another user, you can become that user.
- **Passing a powerful role to something you control.** If you can launch a VM or function and attach a more privileged role to it, then run your code on it, you now act as that role. (In AWS this is the infamous `iam:PassRole` permission.)
- **Changing a trust policy.** If you can edit who's allowed to assume a privileged role, you can add yourself.
- **Impersonation permissions.** In GCP, permissions that let one service account generate tokens for another let you become the other one.

Two takeaways:

1. **"Read-only plus one dangerous permission" can equal admin.** When assessing a compromised identity's blast radius, the question isn't just "what can it do directly?" but "what can it turn itself into?"
2. **IAM changes are high-value detections.** Creating keys, editing policies, modifying trust relationships, and attaching roles are rare relative to normal activity, and they're how attackers escalate and persist.

## Least privilege, and why it never quite happens

The principle of **least privilege** says every identity should have only the permissions it needs. Everyone agrees with it. Almost no environment achieves it. Why?

- It's hard to know in advance what something needs, so people grant broad permissions to make it work.
- Needs change, and permissions are added but rarely removed.
- Tightening permissions risks breaking production, so it's always lower priority than shipping.

The result is **permission sprawl**: identities with far more access than they use. Tools exist that compare granted permissions with permissions actually used (based on logs) and suggest reductions. D&R benefits directly from this work, since every unused permission is attack surface. It's a good example of D&R feeding prevention (first guide, Chapter 7).

## Just-in-time and break-glass access

Two patterns you'll hear about:

- **Just-in-time (JIT) access:** engineers don't hold standing access to sensitive environments. They request it for a specific task and time window, often with approval, and it expires automatically. This shrinks the window in which a stolen session is useful and creates a clear record ("Alice requested production admin for incident 123 at 14:02").
- **Break-glass access:** emergency credentials for when normal access paths fail (for example, the identity provider is down). They're kept locked away, and any use triggers an alert.

For D&R, JIT is a gift: access to sensitive systems without a matching request record is immediately suspicious.

## Mapping back to your background

Most of what you know about account security applies here, but with a twist in emphasis:

- In consumer account security, you mostly ask "is this the real account owner?"
- In cloud IAM, you ask that *and* "what can this identity reach, and what can it turn itself into?"

The second question is where the blast radius lives, and it's where a lot of cloud D&R reasoning happens.

## Questions to think about

1. An alert says "role `deploy-prod` created a new access key for user `svc-backup`." What's the first question you'd ask about the identity chain behind `deploy-prod`?
2. Why are non-human identities often more dangerous than human ones, even though humans are the ones who get phished?
3. An identity has read-only access everywhere, plus permission to update one Lambda function that runs as an admin role. How much access does it really have?
