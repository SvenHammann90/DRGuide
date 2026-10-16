# Chapter 9 — Pipelines and infrastructure as code

In a mature cloud environment, humans rarely change production by hand. Code and infrastructure changes flow through automated pipelines. That's good for security in many ways, but it also concentrates enormous power in the pipeline itself. This chapter explains how that works and why the pipeline is a crown jewel.

## Infrastructure as code

**Infrastructure as code (IaC)** means describing infrastructure in text files and letting a tool create it. Instead of clicking through the cloud console to create a bucket, a role, and a cluster, an engineer writes a file describing them and runs a tool that makes reality match the file.

Common tools:

- **Terraform** (and its open-source fork OpenTofu): the most widely used, works across providers.
- **Provider-specific tools:** AWS CloudFormation and CDK, Google's Infrastructure Manager, Azure's Bicep.
- **Pulumi:** infrastructure defined in normal programming languages.
- **Kubernetes YAML** (with tools like Helm and Kustomize): the same idea for cluster objects.

Benefits for D&R:

- **A record of intent.** The repository says what infrastructure *should* exist, who changed it, when, and why (in the commit and code review).
- **Review before change.** Changes go through code review, so a second person sees them.
- **Reproducibility.** Infrastructure can be rebuilt from the files after a compromise, instead of trusting cleaned-up systems.
- **Drift detection.** Anything in the real environment that doesn't match the files is either a manual change, a mistake, or an attacker.

That last point is one of the most useful ideas in cloud D&R. **If your environment is managed by IaC, then a change made outside IaC is an anomaly by definition.** An IAM role created by a human in the console, in an account where all roles come from Terraform, deserves a question.

One caveat: IaC tools keep a **state file** recording what they've created, and state files often contain secrets (database passwords, keys) in plain text. Where state files are stored, and who can read them, is a security question in itself.

## CI/CD: the pipeline

**CI/CD** (continuous integration / continuous delivery) is the automated pipeline that takes code from a repository, builds it, tests it, and deploys it. Common systems: GitHub Actions, GitLab CI, Buildkite, CircleCI, Jenkins, and cloud-native ones like Google Cloud Build.

A typical flow:

1. An engineer opens a pull request.
2. CI runs tests on the proposed change.
3. After review and merge, the pipeline builds artifacts (container images, packages).
4. The pipeline pushes images to a registry.
5. The pipeline (or a GitOps tool) deploys the new version to production and applies infrastructure changes.

To do steps 3 to 5, the pipeline needs **powerful credentials**: permission to push images, change infrastructure, and deploy to production. Often the deploy identity can change almost anything in the environment, because it has to.

That's why the pipeline is a crown jewel: **whoever controls the pipeline controls what runs in production**, and holds the pipeline's credentials. An attacker who can modify a workflow file, inject code into a build step, or steal a deploy credential doesn't need to break into production. Production will accept their changes as legitimate.

## How pipelines get attacked

**Stolen pipeline secrets.** Long-lived cloud keys stored as CI secrets are a classic target. They can leak through build logs, through compromised build steps, or through anyone who can edit a workflow and print them.

**Malicious workflow changes.** If someone can change the pipeline definition (a workflow file in the repository) and have it run with production credentials, they can do anything. Good setups require review for workflow changes and only give production credentials to workflows running from protected branches after merge.

**Untrusted code running with trusted credentials.** A classic mistake: running CI on pull requests from outside contributors (or from forks) with access to secrets. The pull request's code runs in the pipeline, and can read the secrets. Code hosts have specific settings and pitfalls around this, and real compromises have happened through it.

**Compromised third-party build steps.** Pipelines often use shared, third-party components (GitHub Actions from the marketplace, for example). If one of those is compromised, every pipeline that uses it runs the attacker's code with its own credentials. There have been real incidents in which a popular third-party action was modified to dump secrets from every pipeline that ran it. Pinning third-party components to exact, reviewed versions reduces this risk.

**Dependency poisoning.** Malicious packages pulled during the build, through typosquatting, compromised maintainers, or dependency confusion (an attacker publishing a public package with the same name as your internal one). The build then includes attacker code in what ships.

**Compromised build infrastructure.** The machines that run builds (CI runners). Self-hosted runners that persist between jobs can let one job's attacker affect the next.

## Keyless CI: OIDC federation

Chapter 3 mentioned OIDC federation as the way to remove long-lived keys from CI. Here's the mental model in a bit more detail, because it's both a big improvement and a place where subtle misconfigurations live.

1. When a CI job runs, the CI platform issues it a signed identity token. The token contains **claims** about the job: which organization and repository it's from, which branch, which workflow, whether it's from a pull request, and so on.
2. In the cloud, a role is configured to trust tokens from that CI platform, but **only if the claims match conditions**: for example, "repository must be `ourorg/infra` and branch must be `main`."
3. The job presents its token to the cloud, the cloud checks the signature and the conditions, and if everything matches, it issues short-lived credentials for the role.

No secret is stored anywhere. Compromising the CI secret store gains nothing, because there's nothing in it.

The risk moves to the **conditions**. If the trust policy checks only that the token comes from the CI platform, without checking the repository, then *any repository on that platform* could assume your role, including one an attacker creates. If it checks the repository but not the branch, anyone who can push a branch can deploy to production. Reviewing these conditions is a high-value check, and changes to them are high-value detections.

## Software supply chain security

Beyond your own pipeline, there's the broader question of trusting the software you build from and run. Concepts you'll hear about:

- **SBOM (Software Bill of Materials):** a list of every component in a piece of software, so you can answer "are we affected?" when a vulnerable or malicious package is announced.
- **Artifact signing:** builds produce signed artifacts (container images, packages), and deployment only accepts artifacts with a valid signature from your build system. Tools like Sigstore and cosign are common.
- **Provenance and SLSA:** records of how and where an artifact was built, so you can verify it came from your pipeline and your source, not from someone's laptop. SLSA is a framework of levels for how strongly this is guaranteed.
- **Admission policies for images:** Kubernetes only runs images that are signed and come from approved registries (Chapter 6).

The idea behind all of them is the same: make it possible to verify, at deploy time, that what's running is what you built from what you reviewed.

## D&R in the pipeline

Things worth watching:

- **Changes to pipeline definitions,** especially those that touch credentials, deploy steps, or third-party components.
- **Changes to OIDC trust policies** and to roles the pipeline assumes.
- **Pipeline credentials used from outside the pipeline.** If a deploy role is assumed from an address that isn't your CI provider, something is very wrong.
- **Deploys that didn't come from the pipeline.** Production changes made by human identities, or by unusual identities, in an environment where everything normally comes through CI.
- **Unusual pipeline behavior:** a build that makes unexpected network connections, a job that runs much longer than usual, or a job that reads secrets it doesn't normally read.
- **Code host audit logs:** new deploy keys, new personal access tokens, changes to branch protection, new third-party app installations, and repository visibility changes (a private repository made public).

And the playbook question every D&R team needs an answer to: **a popular third-party component we use was just reported as compromised. What do we do?** Typically: find every pipeline and system that used it during the affected window, assume their secrets were exposed, rotate them, and review what those pipelines' credentials did in that period.

## Questions to think about

1. Why might an attacker prefer to compromise your CI pipeline rather than your production cluster directly?
2. A cloud role trusts GitHub Actions tokens from your organization, with no condition on repository or branch. Describe how an attacker could abuse this, and what you'd look for in the logs.
3. Your environment is fully managed by Terraform. What kinds of legitimate changes would still happen outside Terraform, and how would you avoid drowning in drift alerts?
