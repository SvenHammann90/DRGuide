# Chapter 11 — Two incidents, start to finish

This chapter tells two realistic but fictional incidents as stories, from first signal to closure. The aim is to show how the concepts from the previous chapters fit together in practice, and to show the kinds of decisions a D&R engineer makes along the way. Read them slowly, and at each decision point ask yourself what you would do before reading on.

## Incident 1: The key in the old repository

### The signal

It's a Tuesday morning. An alert arrives from the code host's secret scanning service, via the cloud provider: an AWS access key starting with `AKIA` was found in a public repository. The repository belongs to a former contractor's personal account and contains a copy of an internal deployment script from two years ago.

At almost the same moment, the cloud provider's automated response has attached its quarantine policy to the key, restricting some actions.

### Triage

The on-call engineer works through the first questions:

- **Which identity does the key belong to?** An IAM user called `svc-deploy-legacy`, in the company's main production account.
- **Is the key active?** Yes. Nobody had deleted it.
- **What permissions does that user have?** Broad ones. It was created years ago for an old deployment system and has a policy that allows most actions on compute, storage, and IAM.
- **Has it been used recently?** This is the critical question. The cloud console shows the key was last used **forty minutes ago**. CloudTrail shows the calls came from an IP address belonging to a hosting provider the company doesn't use.

This isn't a hygiene finding anymore. It's an incident. The engineer declares one, takes the incident commander role for now, and pulls in a second engineer to investigate.

### The containment decision

The obvious move is to deactivate the key immediately. But first, a quick check: does anything legitimate still use this key? If a production system depends on it, deactivating it could cause an outage.

The investigator looks at the key's usage over the last 90 days in CloudTrail. Before today, the key hadn't been used in fourteen months. Nothing depends on it. That makes the decision easy:

1. **Deactivate the key** (not delete, so the record stays intact).
2. **Attach an explicit deny-all policy** to the `svc-deploy-legacy` user, which also blocks any temporary credentials issued from it.
3. **Check whether the key was used to assume any roles,** because role sessions wouldn't be blocked by the user's deny policy.

Total time from alert to containment: about fifteen minutes. In this case containing first was clearly right: the attacker was opportunistic (a scanner picking up a public key), the key had no legitimate use, and every minute of delay was more risk.

### Scoping

Now the important question: **what did the attacker do in those forty minutes?** The investigator pulls every CloudTrail event for the key since it appeared in the public repository. The story:

1. **`GetCallerIdentity`** from the hosting provider's IP, using a Python scripting library as the user agent. The attacker checking what they had.
2. **A burst of list calls** across many services. Many succeeded. Some failed with access denied.
3. **`DescribeRegions`** followed by attempts to launch large GPU instances in three regions the company doesn't use. Two succeeded. Classic crypto-mining.
4. **`CreateUser`** for a new user named `backup-admin`, then **`CreateAccessKey`** for it, then **`AttachUserPolicy`** with the built-in administrator policy. **Persistence.**
5. **`ListBuckets`**, which returns the names of every bucket in the account. This is a management event, so it's always recorded. What happened next inside the buckets depends on data events, which are enabled on only some of them. On those, the trail shows the attacker listing objects in two buckets. On the others, the trail can't say whether the attacker listed or downloaded anything.

The investigator also checks the role assumption question from containment: the key was not used to assume any roles. Good.

### Eradication

Working from the scope:

- **Delete the `backup-admin` user and its key.** Before deleting, check whether *that* key was used, and from where. (It was used once, for a `GetCallerIdentity` call, from a different IP. The attacker was testing their backdoor. No other activity.)
- **Terminate the crypto-mining instances** in the unused regions, after taking snapshots for evidence.
- **Search for anything else created** by either key: other users, roles, functions, trust policy changes, security group changes. Nothing else found.
- **Add a guardrail** blocking the unused regions entirely, since the attacker's use of them shows they're a real gap.

### The data question

Legal asks the question that always comes: **did the attacker access customer data?**

The honest answer requires care:

- For buckets with data events enabled, the investigator can say definitively: the attacker listed objects in two of them but downloaded nothing.
- For buckets without data events, CloudTrail alone can't answer the question.
- Other evidence helps: the sensitive buckets are encrypted with a customer-managed KMS key (without the caching feature from Chapter 7), and KMS calls are recorded as management events. The KMS logs show no decrypt calls from the compromised identities. Downloading the objects would have required decrypts. That's strong evidence that no objects were read.

The final statement: "The attacker listed our buckets and the contents of two of them. Based on encryption key logs, which would have recorded any read of these objects, we have high confidence no customer data was downloaded." That's an evidence-based claim, with the reasoning shown, instead of "we found no evidence of data access."

### The postmortem

Root causes and contributing factors:

- A long-lived key for a deprecated system was never deleted.
- The key had far broader permissions than it needed.
- A contractor copied internal code to a personal repository.
- There was no guardrail blocking unused regions.
- Data events weren't enabled on all sensitive buckets.

Action items: find and remove all unused long-lived keys (and alert on any key unused for 90 days), block creation of new IAM user keys by guardrail, enable data events on all buckets tagged sensitive, block unused regions, and add a detection for "new IAM user created outside IaC."

Notice that most of the actions are prevention, not detection. That's typical.

## Incident 2: The shell in the inference pod

### The signal

On a Thursday evening, the runtime security tool raises an alert: **a shell process started inside a production pod** belonging to an internal service that converts uploaded documents into text. The parent process is the document converter itself.

### Triage

- **Is someone debugging?** The investigator checks the Kubernetes audit log for an `exec` into the pod around that time. None. The shell was spawned by the application, not by a human using `kubectl exec`.
- **What does the service do?** It accepts files from other internal services, which in turn accept files from customers. So it processes untrusted input.
- **What else happened in the pod?** The runtime tool shows the shell ran a few commands: listing the filesystem, printing environment variables, and reading the service account token file.

A shell spawned by an application that processes untrusted files, followed by credential reconnaissance, is almost certainly exploitation. Incident declared.

### The containment decision

This one is harder than Incident 1:

- The attacker is active right now, and has a foothold *inside* the cluster.
- Killing the pod is easy, but Kubernetes will just start a new one, and if the vulnerability is in the converter, the attacker can exploit it again.
- Killing the pod also destroys evidence in its memory and filesystem.
- The service is used by other services, so taking it offline has business impact.

The incident commander makes a sequence of decisions:

1. **Capture evidence first,** quickly: the runtime tool's records, the pod's filesystem if possible, and the full audit log trail for the pod's service account.
2. **Isolate rather than kill:** apply a network policy that cuts the pod (and its siblings) off from everything except what's needed for evidence collection. This stops further movement and any outbound channel.
3. **Revoke what the attacker may hold:** the service account token (by deleting and recreating the service account, which invalidates its tokens), and any cloud credentials already issued to it through workload identity (by revoking active sessions for the linked cloud role, since those short-lived credentials would otherwise stay valid until they expire).
4. **Scale the service down** temporarily, accepting the business impact, until the vulnerability is understood. Tell the dependent teams.

### Scoping

The questions follow the attack path from Chapter 6:

- **What could the pod's service account do in Kubernetes?** The audit log shows the token was used: the attacker asked "what can I do?" and then tried to list Secrets in the namespace. RBAC denied it. The service account had no API permissions beyond reading its own ConfigMap. Good: the escalation path through the Kubernetes API was closed.
- **Was the pod privileged, or did it have host mounts?** No. Admission policy required the restricted profile. No sign of escape to the node.
- **What cloud identity was linked to the pod?** A role with permission to read and write one bucket, used to store converted text. The cloud audit log shows the role's credentials being used from the pod (expected) and, ten minutes after the shell started, **from an external IP address** (not expected). The attacker took the credentials out of the cluster.
- **What did they do with the role?** `GetCallerIdentity`, then attempts to list other buckets (denied), then reads on the one bucket the role could access. Data events are enabled on that bucket, so the investigator can list exactly which objects were read: about two hundred converted documents.
- **Network:** DNS and egress logs show the pod connected to an external server shortly after the shell started. That was the attacker's command channel. The pod's egress wasn't restricted, which is how the shell could talk out.

### Eradication and recovery

- The vulnerable component turns out to be a file-parsing library with a known vulnerability. The team updates it, rebuilds the image, and redeploys.
- The service's role gets a condition restricting its use to the cluster's network, so credentials taken out of the cluster won't work elsewhere.
- An egress policy is added: the converter has no reason to reach the internet at all.
- The service is restored and watched closely for repeat attempts.

### The data question, again

This time the answer is precise, because data events were enabled: the attacker read a specific list of two hundred documents. That list goes to legal and the relevant teams, who work out whose data was in them and what notification obligations follow.

### What went well, and what didn't

Went well: the runtime detection caught the shell within seconds. Tight RBAC and admission policy stopped the attacker from escalating inside the cluster. Data events on the bucket made scoping precise.

Didn't go well: the library vulnerability had been known for weeks. The pod could reach the internet for no reason. The role's credentials worked from anywhere.

The pattern from Incident 1 repeats: **detection found it, but prevention decided how bad it got.**

## What the two stories have in common

- The first questions were always about **identity**: which credential, which principal, what can it do?
- The scope came from **audit logs**, and the gaps in scope came from **missing logs**.
- Containment involved real **trade-offs**, made explicitly.
- Answering "what data was taken?" depended entirely on decisions made long before the incident (data events, KMS keys).
- Most of the action items were **prevention**.

That's what most cloud incidents look like.

## Questions to think about

1. In Incident 1, what would you have done differently if the key had been used by a critical production system every few minutes?
2. In Incident 2, why delete and recreate the Kubernetes service account instead of just deleting the pod? What would have happened if you'd only restarted the pod?
3. Pick one of the two incidents and list which single prevention control, had it been in place, would have reduced the impact the most.
