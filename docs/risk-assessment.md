# 📋 Risk Assessment: Shared Long-Term IAM Credentials

## Incident Summary

A long-term IAM access key ("the installation key") used by Cloudify for CICD operations on Amazon EKS was:

1. Extracted from Cloudify by an SRE
2. Shared among operations team members
3. Published on the company's Confluence page (Key ID + Secret in plaintext)

## Root Cause

Operations engineers needed access to AWS resources to perform their daily tasks but were not provided an alternative authentication method. They created a workaround out of necessity.

## Security Risks

| **Risk**                | **Severity** | **Description**                                                                                                  |
|-------------------------|--------------|------------------------------------------------------------------------------------------------------------------|
| Credential Exposure     | 🔴 Critical  | Key ID and Secret visible to anyone with Confluence access (hundreds of employees)                               |
| No Audit Trail          | 🔴 Critical  | Shared key = no individual accountability. CloudTrail logs show the service user, not the actual person          |
| Single Point of Failure | 🔴 Critical  | One key serves the entire CICD infrastructure for 100+ development teams                                         |
| Lateral Movement        | 🟠 High      | Key permissions likely broader than needed — attacker could pivot to other services                              |
| Compliance Violation    | 🟠 High      | Long-term credentials shared in plaintext violates security policies and compliance frameworks (SOC2, ISO 27001) |
| External Leak           | 🔴 Critical  | If the key leaks beyond the organization: full infrastructure compromise                                         |

## Business Impact

| **Impact**            | **Severity** | **Description**                                                                         |
|-----------------------|--------------|-----------------------------------------------------------------------------------------|
| Application Outage    | 🔴 Critical  | Business-critical CICD system serving 100+ teams could be disrupted or destroyed        |
| Infrastructure Damage | 🔴 Critical  | Irreversible damage to AWS infrastructure (resource deletion, configuration corruption) |
| Data Leakage          | 🔴 Critical  | Sensitive build artifacts, source code references, configuration secrets exposed        |
| Reputation Damage     | 🟠 High      | Loss of trust from development teams and management                                     |
| Financial Impact      | 🟠 High      | Unauthorized resource creation, data transfer costs, incident response costs            |

## Overall Severity

> *CRITICAL — Immediate action required with zero service disruption.*

## Constraints on Remediation

1. No disruption — Key rotation must not affect the running CICD system
2. No blocking — Operations engineers must have alternative access before old access is removed
3. Political sensitivity — Avoid friction with the cybersecurity team
4. Parallel execution — All workstreams must run simultaneously
