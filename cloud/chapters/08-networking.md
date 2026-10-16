# Chapter 8 — Networks in the cloud

Cloud networking matters less to D&R than it did in the days of the corporate network perimeter, because identity has taken over as the main control. But it still matters in two specific ways: it decides what an attacker can *reach* once they're inside, and it decides how data can *leave*. This chapter covers enough to reason about both.

## The building blocks

**VPC (Virtual Private Cloud).** A private network inside the cloud, isolated from other customers. Resources like VMs, databases, and Kubernetes nodes live inside a VPC and get private addresses. (Azure calls this a VNet.)

**Subnets.** Subdivisions of a VPC, usually split into:
- **Public subnets:** resources here can have public internet addresses and be reached from the internet (subject to firewall rules). Load balancers typically live here.
- **Private subnets:** resources here have no public address. They can't be reached from the internet directly. Most servers, databases, and clusters should live here.

**Firewall rules.** AWS calls them **security groups** (attached to resources) and network ACLs (attached to subnets). GCP calls them firewall rules. They say which traffic is allowed in and out: "allow port 443 from the load balancer," "allow the database port only from the application servers."

**Internet and NAT gateways.** An internet gateway lets public resources talk to the internet. A **NAT gateway** lets private resources make *outbound* connections to the internet (to download updates, call external APIs) without being reachable from it. Every outbound connection from private resources goes through it, which makes it a natural place to watch and control egress.

**Load balancers.** The front door for internet traffic to your services. They receive requests and forward them to servers or pods inside private subnets.

**Peering and private connectivity.** VPCs can be connected to each other (peering, transit gateways), to on-premises networks (VPN, dedicated links), and to cloud services without crossing the internet (**private endpoints**, called VPC endpoints in AWS and Private Service Connect in GCP).

## What an attacker can reach

Once an attacker has a foothold on a machine or pod, the network decides where they can go next. In a **flat network**, where everything can talk to everything, a compromised web server can connect to the database, to internal admin tools, to other teams' services, and to Kubernetes nodes. In a **segmented network**, it can reach only what it needs.

Segmentation tools at different layers:

- **Separate VPCs or accounts** for very different environments (production vs development, or sensitive systems vs general ones).
- **Subnets and firewall rules** within a VPC.
- **Kubernetes network policies** within a cluster (Chapter 6).
- **Service meshes** (like Istio), which can require every service-to-service connection to be authenticated and authorized, so even reachable services refuse unknown callers.

A useful D&R habit: for any system that gets compromised, ask **"what could it reach on the network?"** alongside "what could its identity do?" Both determine the blast radius.

## Zero trust, briefly

You'll hear **zero trust** often. Stripped of marketing, it means: **don't grant access based on network location.** Being "inside the corporate network" or "on the VPN" shouldn't be enough to reach anything. Every request should be authenticated (who are you?), authorized (are you allowed?), and ideally checked against device health (is this a managed, healthy device?).

In practice, this often looks like **identity-aware proxies** in front of internal tools (Google's BeyondCorp is the origin of the idea, and GCP's IAP is a product version of it). Engineers reach internal apps through the proxy, which checks their SSO identity and device before letting them in, rather than through a VPN that puts them "on the network."

For D&R, zero trust shifts the important telemetry from network logs to the proxy and identity provider logs: who accessed which internal tool, from which device. It also means a stolen VPN credential is worth less to an attacker, and a stolen session or compromised managed device is worth more. That pushes the threat model back toward identity and endpoints, which is where the first guide spent most of its time.

## Egress: the part D&R cares about most

Inbound traffic gets most of the attention in network security, but for D&R, **outbound traffic (egress)** is often more important, for two reasons:

1. **Command and control.** Malware and attackers' tools need to talk to the attacker. If a compromised pod can connect anywhere on the internet, the attacker has an easy channel.
2. **Exfiltration.** Data that leaves through a workload, rather than through the cloud API, leaves over the network.

Most production workloads only need to talk to a small set of destinations: other internal services, a few cloud APIs, maybe a handful of external services. Restricting egress to an **allowlist** of known destinations is one of the most effective controls against both C2 and exfiltration, and one of the least commonly done, because building and maintaining the list takes effort.

Ways egress is controlled:

- **No internet route at all** for the most sensitive systems. They reach cloud services only through private endpoints.
- **Egress proxies or firewalls** that only allow approved domains.
- **Private endpoint policies** that restrict which buckets or resources can be reached through the endpoint (for example, "only buckets in our organization"), which blocks copying data to an attacker's bucket over the private path.
- **Bandwidth limits.** For extremely valuable, extremely large assets (model weights are the obvious example), limiting the outbound bandwidth of systems that hold them means that stealing them would take a long time, and a long, sustained transfer is much easier to notice and stop. Anthropic has publicly mentioned egress bandwidth controls as part of its security measures for protecting model weights.

## Network telemetry

The main network sources in cloud D&R:

- **Flow logs** (VPC Flow Logs in AWS and GCP): records of connections, meaning source and destination addresses, ports, and byte counts, but not content. Useful for spotting unusual destinations, large transfers, and connections that shouldn't exist (a database talking to the internet). Volume is high, so they're often sampled or kept briefly.
- **DNS logs:** which domain names your resources looked up. Very useful, because almost all outbound connections start with a DNS lookup, and DNS logs are much smaller than flow logs. A pod looking up a domain registered last week, or a known mining pool, is a strong signal.
- **Proxy and firewall logs:** if egress goes through a proxy, it records which domains were contacted and how much data was sent.
- **Load balancer and web application firewall logs:** inbound requests, useful for spotting exploitation attempts against your applications.

A good mental model: **network logs tell you *where* and *how much*, not *who* or *why*.** To answer who and why, you join them with identity and workload information: which pod had that IP address at that moment, running which service, with which identity. In environments where addresses are reused constantly (Kubernetes, again), that join is a real data engineering problem.

## Questions to think about

1. A pod that normally only talks to two internal services starts making DNS lookups for a domain you've never seen. What hypotheses would you consider, and what would you check first?
2. Why might an egress allowlist be more valuable than an inbound firewall for a system that holds model weights?
3. In a zero trust setup without a VPN, what does an attacker need to reach an internal tool, and where would you see them?
