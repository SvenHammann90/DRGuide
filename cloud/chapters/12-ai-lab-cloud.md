# Chapter 12 — The AI lab cloud

Everything so far applies to any company running in the cloud. This chapter applies it to the specific shape of a frontier AI lab's infrastructure. As in the first guide, it's based on public information and general industry practice, not on knowledge of Anthropic's internal architecture. Treat it as a way to reason about the environment, and as a source of good questions to ask in conversations with the team.

## What's publicly known about the shape

Anthropic has publicly announced large compute partnerships with several cloud providers: Amazon Web Services (including AWS's Trainium chips and the large "Project Rainier" cluster), Google Cloud (including access to large numbers of TPUs), and Microsoft Azure. Its products are also offered through those providers' platforms.

You don't need to know the details to draw the important conclusions:

- **The environment is almost certainly multi-cloud.** That means multiple IAM systems, multiple audit log formats, multiple Kubernetes flavors, and multiple sets of native security tools. Normalizing telemetry across providers (first guide, Chapter 3) and reasoning about identity that spans them is likely a big part of the work.
- **The scale is very large.** Large accelerator clusters, enormous datasets, high-volume inference. Large scale means automation everywhere, a lot of non-human identities, and large log volumes.
- **Some infrastructure is specialized.** Accelerator clusters (GPUs, TPUs, Trainium) have their own hardware, drivers, schedulers, and networking, and some of it differs from standard web infrastructure.

## The parts of an AI lab's cloud

A rough map of what a frontier lab runs, and why each part matters for security:

**Training clusters.** Very large groups of accelerator machines working together on a single training run, connected by high-speed networks. They're scheduled either with Kubernetes or with HPC schedulers like **Slurm** (common in research computing, and with a different security model: users typically log into shared "login nodes" and submit jobs). Training jobs read huge datasets and write **checkpoints**: periodic saves of the model's weights. Those checkpoints *are* model weights.

**Data pipelines and datasets.** Collection, cleaning, and storage of training data, mostly in object storage and data warehouses. Datasets are large and valuable, and the integrity of data matters as well as its confidentiality: tampered training data could affect the model.

**Model weights storage.** Final and intermediate weights, stored in object storage, likely in tightly controlled accounts with customer-managed encryption keys, restricted access, and data perimeters.

**Evaluation and research infrastructure.** Systems that run tests on models, including safety evaluations. Researchers need flexible access to models and compute here.

**Inference serving.** The systems that run models to answer customer requests. To serve a model, its weights have to be loaded onto serving machines. That means weights exist not just in a vault but on every serving node, in memory, across regions and potentially across providers.

**Research and development environments.** Notebooks, remote dev boxes, and GPU machines for experimentation. As Chapter 4 noted, these combine broad access with unpredictable behavior.

**Corporate and product infrastructure.** Everything a normal tech company runs: the website, the API platform, billing, customer data, internal tools, CI/CD, SaaS.

**Internal AI agents.** Labs use AI agents heavily for engineering and research work. These agents hold credentials and act in the environment, which makes them a new and important kind of non-human identity.

## Model weights: following the asset

The first guide described model weights as a top crown jewel. With the cloud concepts from this guide, you can reason about protecting them more concretely. A good exercise: **follow the weights through their whole lifecycle and ask where they exist, who can reach them, and what you'd see.**

1. **During training,** weights live in the memory of training nodes and are written out as checkpoints. Who can access training nodes? Who can read the checkpoint storage? Can a training job's identity write checkpoints somewhere else?
2. **At rest,** checkpoints and final weights sit in storage. Which accounts? Which identities can read them? Encrypted with which key, and who can use that key? Is there a data perimeter? Are data access events and key usage logged?
3. **When moved,** weights are copied between storage, evaluation systems, and serving systems, possibly across regions or providers. Through which paths? Using which identities? Is each transfer expected and recorded?
4. **During serving,** weights are loaded onto inference machines. Who can access those machines, run code on them, or debug them? Can the machines send data anywhere except where they need to?
5. **In research use,** do researchers ever have direct access to weights, or only through controlled interfaces? What about internal AI agents?

At every stage, the defensive themes from earlier chapters apply:

- **Minimize the number of copies and the number of paths.** Every copy is attack surface.
- **Minimize the number of identities with access,** and require multiple people for the most sensitive operations. Anthropic has publicly described multi-party authorization requirements as part of its security measures around weights.
- **Use the size of weights.** Frontier models are large, often hundreds of gigabytes or more. Stealing them requires moving a lot of data, which is slow if egress bandwidth is limited and noticeable if egress is monitored. This is why egress controls (Chapter 8) are so central to weight protection.
- **Log access exhaustively** for the systems that hold weights, even if it's expensive. This is the one place where "log every data access" is clearly worth it.

Anthropic has also written publicly about exploring confidential computing for inference, meaning hardware-based protections that keep weights encrypted even from the machine's own operators. That's an example of how weight protection pushes beyond standard cloud controls.

## Accelerators and their particular risks

Accelerator infrastructure brings some specific considerations:

- **Valuable to opportunistic attackers too.** GPU instances are exactly what crypto-miners want, and stolen cloud credentials are often used to launch them (Chapter 11, Incident 1). Unexpected GPU instance creation is a good detection anywhere, and an AI lab has a lot of legitimate GPU usage to blend into.
- **Specialized software stacks.** Drivers, firmware, and high-performance networking software are a large and less-scrutinized attack surface, and patching them across large clusters is operationally hard.
- **Shared clusters.** Many teams and jobs share accelerator clusters. Isolation between jobs, especially between jobs with different sensitivity, matters (Chapter 6, the multi-tenant problem).
- **Different schedulers, different logs.** If part of the fleet runs on Slurm rather than Kubernetes, its access model and telemetry differ, and detection coverage has to be built for both.

## Why baselining is hard here

The first guide warned that "unusual is not malicious." At an AI lab, this is especially true:

- Researchers legitimately move very large amounts of data.
- New projects start constantly, with new access patterns.
- Infrastructure is growing fast, so "normal" keeps changing.
- Internal AI agents generate activity that doesn't look like any human's.

That pushes detection strategy toward the approaches this guide has emphasized: **define the approved paths for the most sensitive actions, and detect deviations from those paths,** rather than trying to model "normal" for everything. For example, rather than detecting "unusual data movement" in general, define "weights only move through these pipelines, run by these identities" and alert on anything else touching weight storage.

## AI agents in the cloud

Internal AI agents deserve a final word, because they combine several themes from this guide:

- They're **non-human identities** (Chapter 2) with potentially broad permissions.
- They may hold **credentials** (Chapter 3), and their environments are places where credentials can be found.
- They read **untrusted content** (code, documents, web pages, logs), which can contain prompt injection attempts to steer them (first guide, Chapter 9).
- Their behavior is **less predictable** than a classic service account's.

The defensive patterns are familiar: least privilege, short-lived credentials, sandboxed execution environments, egress restrictions, approval steps for sensitive actions, and detailed logging of what agents do. The detection problem is newer: distinguishing an agent doing its job from an agent that's been manipulated, or whose credentials have been taken. This is an open area, and having thought about it is a good sign in conversations about the role.

## Good questions to ask

If you get the chance to talk with people on the team, questions like these show you've thought about the environment (and you'll learn a lot from the answers):

- How do you think about visibility across multiple cloud providers? Do you normalize everything into one platform?
- Where do the hardest attribution problems come up: role chains, cross-cloud identity, agents?
- How much of your detection is built around "approved paths" for sensitive assets vs behavioral baselines?
- How do you approach detection for internal AI agents?
- What does the balance look like between building detections and pushing for prevention?

## Questions to think about

1. Follow model weights from a training checkpoint to an inference server. List every place they exist along the way, and for each, the one control you'd most want.
2. Why might "approved path" detection work better than anomaly detection for weight storage at an AI lab? What would it miss?
3. An internal AI agent's cloud credentials are used to read from a bucket the agent has never accessed. List benign and malicious hypotheses, and the evidence that would separate them.
