# 📖 The Story — Context Behind the Walkthrough

This is the real-world incident the rest of this repository is built around. The
hands-on guides (cluster setup, key rotation, permission sets, RBAC, access
config) all exist to reproduce and remediate what's described here. Read this
first for the "why"; the [walkthrough](../README.md#-technical-walkthrough) is
the "how".

> All names, values, and identifiers elsewhere in the repo are placeholders. This
> account is anonymized.

---

## Context

### The team

I worked as an **AWS Security Architect** in a global, multicultural
organization. The team ran the daily operations of a system hosting CICD
pipelines for **100+ development teams**. We were spread across three time zones
(North America, EMEA, India): three architects, first-line operations engineers,
SREs, a scrum master, a program manager, and an IT operations manager. The team
sat inside the IT organization, in a department supporting embedded-software
developer teams.

### The system

The CICD platform ran on **AWS, on an Amazon EKS cluster**, and was made up of
many modules — pipeline management, identity and resource management, Linux image
builds, IoT components, and more — with Prometheus and Grafana as an internal
monitoring layer.

The whole system was deployed and managed with **Cloudify**. At the time,
Cloudify's only authentication option was a **long-term AWS access key attached
to an IAM user** — known internally as **"the installation key."**

### The operating climate

The system was large and hard to maintain. The team was buried in tickets and
escalations, so one hard constraint was baked in from the start: **operations
work could not be interrupted.** The department director cared about security and
about keeping a good relationship with the corporate IT and cybersecurity
functions, and she cared about process.

---

## What was discovered

In a casual conversation with one of the operations engineers, I learned that the
installation key had been **extracted from Cloudify by an SRE** and then **shared
among team members** to do their day-to-day work. Worse, the **key ID and secret
were published on the company's Confluence page** as part of the operations
documentation — visible to anyone with Confluence access.

The root cause wasn't malice. The operations engineers **had a job to do and no
sanctioned way to do it**, so they improvised. The shared key was a workaround
for a missing capability.

---

## Remediation

As the AWS Security Architect, I owned the fix.

### The core task

The immediate instinct — "just rotate the key" — is a four-step dance:

1. Create a new key
2. Update the system configuration to use it
3. Deactivate the old key
4. Delete the old key

But rotation alone doesn't solve the real problem: engineers would still have no
legitimate way to work, and would invent the next workaround. So the task was
actually **two workstreams run in parallel**:

1. **Rotate the installation key** without disrupting the running CICD system.
2. **Give operations engineers a sanctioned way to authenticate** to AWS and to
   the Kubernetes cluster.

### Constraints

- Key rotation must not break the running CICD system.
- Operations engineers must get alternative access **before** the old access is
  removed — no one gets blocked.
- Political: avoid friction with the cybersecurity team, who were not deeply
  familiar with AWS security.

### Workstream 1 — alternative access for engineers

**AWS layer (IAM Identity Center):**

- A dedicated **Entra ID group** for operations engineers, synchronized into
  **AWS IAM Identity Center** (formerly AWS SSO).
- A scoped permission set, **`LimitedWorkingAccess`** (AWS-managed `ReadOnlyAccess`
  plus a tailored inline policy).
- Association of that permission set with the relevant AWS accounts and the group.

**Kubernetes layer (EKS):**

- A **ClusterRole** with limited permissions and a ClusterRoleBinding.
- An **EKS access entry** in each account binding the SSO role to the Kubernetes
  group (with the managed view policy for read access).
- Testing of the access in both the **AWS Management Console** and the **AWS CLI**.

**People layer:**

- Training and documentation on using the new access from the console and the CLI,
  including configuring AWS CLI credentials via SSO.

### Workstream 2 — rotating the installation key

- **CloudTrail analysis** via CloudWatch Logs Insights, repeated several times, to
  understand where and how the key was used.
- Create a new key and **update Cloudify** to use it.
- Test.
- **Deactivate** the old key, confirm via CloudTrail that nothing still uses it,
  then **delete** it.

Everything was done on the **pre-prod environment first**, then production.

> **Surprise:** the installation key turned out to be used by the **embedded
> Grafana** too. It caused a minor disruption, but was easy to fix once found —
> exactly the kind of hidden dependency the CloudTrail analysis is meant to catch.

### The final step — temporary credentials for Cloudify

The end goal was to get Cloudify off long-term keys entirely and onto **temporary
credentials**.

- **Trade-off:** to run Cloudify, an operator had to place their own temporary
  credentials on the EC2 machine where Cloudify ran.
- **Risk:** someone could hijack the EC2 SSH or SSM session and use another
  person's credentials.
- **Risk:** temporary credentials expire after a few hours, which caused real
  interruptions during long upgrades.

The architects' team weighed these and **accepted the risks**, switching Cloudify
to temporary credentials as the final state.

---

## The hardest part wasn't technical

The technical complexity was manageable. The organizational friction was not.

- **Pushback from ITSM and operations engineers:** "You're making our job more
  complicated."
- **Pushback from the lead architect:** "This isn't our job — talk to the
  cybersecurity team."

What made it work:

- **Get the department director on your side** — she was informed immediately.
- **Get the ITSM manager on your side** — he came around after discussion and
  backed the effort.
- **Explain the why**, repeatedly.
- **Be resilient** against constant pushback in team meetings, from two directions.
- **Be patient.**
- **The controversial question:** when (and how) to involve the cybersecurity team.

---

## Where this maps in the repo

| Story beat | Guide |
|------------|-------|
| The system (EKS cluster) | [cluster-setup/eks-and-storage.md](../cluster-setup/eks-and-storage.md) |
| The installation key | [iam-keys/iam-keys-setup.md](../iam-keys/iam-keys-setup.md) |
| CloudTrail analysis | [cloudtrail-analysis/cloudtrail-analysis.md](../cloudtrail-analysis/cloudtrail-analysis.md) |
| `LimitedWorkingAccess` permission set | [access-setup/policies.md](../access-setup/policies.md) · [access-setup/iam-identity-center.md](../access-setup/iam-identity-center.md) |
| Kubernetes ClusterRole + binding | [kubernetes/README.md](../kubernetes/README.md) |
| EKS access entries | [access-setup/eks-access-entries.md](../access-setup/eks-access-entries.md) |
| Console + CLI access (training) | [access-setup/console-access-guide.md](../access-setup/console-access-guide.md) · [access-setup/cli-access-guide.md](../access-setup/cli-access-guide.md) |
| Rotating the key | [iam-keys/iam-keys-rotation.md](../iam-keys/iam-keys-rotation.md) |
| Temporary credentials for Cloudify | [cloudify/temporary-credentials.md](../cloudify/temporary-credentials.md) |
| Risk assessment | [docs/risk-assessment.md](risk-assessment.md) |
| Lessons learned | [docs/lessons-learned.md](lessons-learned.md) |
| Change management | [docs/change-management.md](change-management.md) |
