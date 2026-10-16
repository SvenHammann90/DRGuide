# Inside the Cloud: A D&R Reader's Guide

A follow-up to [From Account Abuse to Detection & Response](../README.md). The first guide explained how D&R thinks. This one goes one level deeper into the environment you'd be defending: the cloud infrastructure that a company like Anthropic runs on.

The goal is a working mental model, not operational skill. After reading it you should be able to:

- Picture how a cloud environment is organized, and where the security boundaries are.
- Understand what "cloud keys" and other credentials actually are, where they live, how they leak, and why revoking them is harder than it sounds.
- Explain what Kubernetes is, why everyone uses it, and how attackers move through it.
- Know which logs show what, and which behaviors are worth detecting.
- Follow a realistic cloud incident from first alert to eviction.

As before, it's written to be read on a commute. No labs, no commands to run. Where real names appear (AWS, Google Cloud, Azure, specific services), they're there so you recognize them in job descriptions and conversations, not so you memorize them. The concepts transfer across providers.

## Chapters

| # | Chapter | What you get out of it |
|---|---------|------------------------|
| 0 | [The cloud mental model](chapters/00-cloud-mental-model.md) | Why everything is an API call, and the map of layers used in the rest of the guide |
| 1 | [Organizations, accounts, and blast radius](chapters/01-organizations-and-accounts.md) | How cloud environments are structured, and why the structure is a security control |
| 2 | [Cloud identity and IAM](chapters/02-cloud-iam.md) | Principals, policies, roles, and how "who can do what" is decided |
| 3 | [Keys, tokens, and secrets](chapters/03-keys-tokens-secrets.md) | The mental model of cloud credentials: kinds, lifetimes, where they live, how they leak, how to revoke them |
| 4 | [Compute: machines, containers, and functions](chapters/04-compute.md) | VMs, the metadata service, containers, and serverless, seen as a defender |
| 5 | [Kubernetes from zero](chapters/05-kubernetes-basics.md) | What Kubernetes is and how it works, explained without assuming any background |
| 6 | [Attacking and defending Kubernetes](chapters/06-kubernetes-security.md) | Attack paths through a cluster, what you'd see, and what prevents them |
| 7 | [Storage, data, and encryption](chapters/07-storage-and-data.md) | Buckets, databases, encryption keys, and the ways data leaves |
| 8 | [Networks in the cloud](chapters/08-networking.md) | VPCs, firewalls, private access, and why egress is the part D&R cares about |
| 9 | [Pipelines and infrastructure as code](chapters/09-pipelines-and-iac.md) | How infrastructure gets changed, and why the pipeline is a crown jewel |
| 10 | [Cloud logging and detection](chapters/10-cloud-logging-and-detection.md) | The audit logs, their blind spots, and a catalogue of behaviors worth detecting |
| 11 | [Two incidents, start to finish](chapters/11-worked-incidents.md) | A leaked CI key and a compromised pod, told as stories |
| 12 | [The AI lab cloud](chapters/12-ai-lab-cloud.md) | GPU clusters, model weights, multi-cloud, and what's distinctive about defending them |
| 13 | [Glossary](chapters/13-glossary.md) | Cloud and Kubernetes terms in plain language |

## A caveat, again

Chapter 12 reasons about what an AI lab's cloud is likely to look like, using public announcements and general industry practice. It doesn't describe Anthropic's actual architecture.
