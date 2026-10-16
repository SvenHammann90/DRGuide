# Chapter 4 — Compute: machines, containers, and functions

Compute is where code runs. For D&R it's where attackers execute commands, where credentials get harvested, and where a foothold turns into lateral movement. This chapter covers the three common forms of cloud compute, each from a defender's point of view, and sets up the Kubernetes chapters that follow.

## Virtual machines

A **virtual machine (VM)** is a computer simulated by the cloud provider on shared physical hardware. AWS calls them EC2 instances, GCP calls them Compute Engine instances, Azure calls them Virtual Machines. From the inside, a VM looks and behaves like a normal Linux or Windows server. You own everything from the operating system up: patching, software, users, configuration.

Things that matter for D&R:

**Images.** VMs start from an **image** (a snapshot of a disk with an operating system and software installed). If the image is trustworthy and the machine is rebuilt from it regularly, compromises don't last long. If people log in and change things by hand over months, every machine becomes unique and hard to reason about.

**Attached identity.** A VM usually has an IAM role (AWS) or service account (GCP) attached. Code running on the VM can use that identity's permissions. This is the main bridge between "I compromised a server" and "I can now call the cloud API." The scope question for any compromised VM is: **what role was attached, and what can that role do?**

**Access to the machine.** How do people get a shell on a VM? Options range from SSH with long-lived keys (riskier, harder to audit) to provider-managed sessions that go through IAM and are logged (AWS Systems Manager Session Manager, GCP's OS Login and IAP). The latter are much better for D&R, because every login is tied to an identity and recorded.

**Host telemetry.** The cloud audit log tells you that a VM was created and which API calls its role made. It does *not* tell you what happened inside the VM: which processes ran, which files were read, which connections were made. For that you need an agent on the machine (an EDR or similar tool, first guide Chapter 3). Without host telemetry, a compromised VM is a black box with an API-calling identity attached.

## The instance metadata service

Every cloud VM can reach a special internal web address, the **instance metadata service (IMDS)**. On AWS and GCP it lives at the link-local address `169.254.169.254`. Code on the VM can ask it things like "what's my instance ID?" and, crucially, "give me credentials for my attached role."

This is how keyless access works for VMs (Chapter 3): the credentials are fetched fresh from the metadata service, never stored on disk.

It's also one of the best-known attack paths in cloud security. Here's how:

1. A web application on the VM has a **server-side request forgery (SSRF)** vulnerability: an attacker can make the server fetch a URL of their choosing. (For example, an "import from URL" feature that doesn't validate what URL it's given.)
2. The attacker asks the application to fetch the metadata service's credentials URL.
3. The server dutifully fetches its own role credentials and returns them to the attacker.
4. The attacker now holds valid temporary credentials for the VM's role, usable from anywhere.

This pattern was central to the 2019 Capital One breach, where an attacker used SSRF to obtain role credentials from a misconfigured server and then downloaded data from storage buckets. It's a textbook example of a small application bug becoming a large cloud breach because of the identity attached to the machine.

Defenses:

- **IMDSv2 (AWS)** and the required metadata header (GCP) make the metadata service demand a special header or token that a typical SSRF can't add. Enforcing IMDSv2 everywhere is standard advice and can be required by guardrail.
- **Least privilege on attached roles,** so that stolen credentials can do little.
- **Detection:** credentials belonging to a VM's role being used *from somewhere other than that VM*. AWS GuardDuty has a built-in finding for exactly this ("instance credentials used from an external IP address"). Conceptually, it's the same as detecting a stolen session cookie being used from a new device.

## Containers

A **container** is a way of packaging an application with everything it needs (code, libraries, configuration) so it runs the same everywhere. Docker made the idea popular.

The key mental model: **a container is not a small VM.** A VM has its own operating system kernel. A container is just a normal process on a host machine, with the kernel giving it an isolated view of the system:

- Its own view of the filesystem (it sees only the files from its image).
- Its own process list (it can't see other processes on the host).
- Its own network interface.
- Limits on how much CPU and memory it can use.

These isolation features (Linux namespaces and cgroups, if you want the terms) are good, but they're weaker than a VM's isolation, because all containers on a host share the same kernel. A kernel vulnerability, or a container started with too many privileges, can let a process **escape** the container and act on the host itself. Once on the host, an attacker can usually reach every other container on it.

Some vocabulary:

- **Image:** the packaged application, built from a recipe (a Dockerfile). Images are built once and run many times.
- **Registry:** where images are stored and pulled from (Docker Hub, AWS ECR, Google Artifact Registry). If an attacker can push a malicious image to your registry, or you pull a malicious public image, they get code execution wherever it runs. That's a supply chain problem (Chapter 9).
- **Container runtime:** the software on the host that actually starts and runs containers.

Things that matter for D&R:

**Containers are meant to be boring.** A well-built container runs one application and does the same things every time. That makes behavioral detection inside containers much more effective than on developer laptops. A web server container that suddenly spawns a shell, downloads a binary, or connects to an unfamiliar address is very likely compromised. **"A shell spawned inside a production container" is one of the highest-signal detections in cloud security.**

**Containers are short-lived.** A container might live for minutes. If it was compromised and then replaced, the evidence went with it, unless runtime telemetry was shipped off the host while it ran.

**Images can contain secrets and vulnerabilities.** Scanning images before deployment catches known vulnerable libraries and accidentally included secrets.

**Privileged settings matter.** A container started in "privileged" mode, or with the host's filesystem mounted inside it, has little real isolation. Reviewing how containers are allowed to start is a key prevention control (Chapter 6).

## Serverless functions

**Serverless** compute (AWS Lambda, Google Cloud Functions and Cloud Run, Azure Functions) runs your code on demand without you managing any machine. You upload code, attach an identity, and the provider runs it when triggered.

For D&R:

- **The attack surface is mostly the code and its identity.** There's no long-running host to compromise, but a vulnerable function can still be exploited, and its credentials (available in environment variables at runtime) stolen.
- **Functions are a persistence and escalation tool.** An attacker who can create or modify a function, and attach a privileged role to it, can run code as that role (the "pass role" pattern from Chapter 2). Watch for function code or configuration changes outside the normal deployment pipeline.
- **Visibility is thinner.** You get the provider's logs and whatever the function itself logs, but no host agent. Detection leans on the control plane (who changed the function) and on the API calls the function's identity makes.

## Developer environments

One more kind of compute matters a lot in tech companies, especially AI labs: **development environments.** Researchers and engineers often work on powerful remote machines (cloud VMs, GPU boxes, notebook servers like Jupyter, or remote development environments) that have broad access to data and infrastructure.

These machines combine the worst properties for D&R: highly privileged access, unpredictable behavior (people run everything), long-lived credentials sitting in config files, and sometimes weaker monitoring than production. They deserve specific attention in any threat model. "An attacker on a researcher's remote dev box" is a scenario worth thinking through, and Chapter 12 comes back to it.

## Questions to think about

1. A web application has an SSRF bug. What determines whether that's a minor issue or a major breach?
2. Why is "a shell spawned in a production container" a much better detection than "a shell spawned on a developer laptop"?
3. Compare a compromised container, a compromised VM, and a compromised serverless function. For each, where would the attacker look for credentials, and what logs would show you what they did?
