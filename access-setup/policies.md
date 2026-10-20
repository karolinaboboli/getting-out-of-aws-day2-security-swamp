# 🔐 IAM Permission Set: LimitedWorkingAccess

## Overview

The LimitedWorkingAccess permission set is designed for operations engineers who need day-to-day access to the CICD infrastructure on AWS. It follows the principle of least privilege — granting only the permissions needed for operational tasks.

## Composition

The permission set consists of two components:

| **Component**        | **Type**             | **Purpose**                                       |
|----------------------|----------------------|---------------------------------------------------|
| ReadOnlyAccess       | AWS Managed Policy   | Baseline read permissions across all AWS services |
| Custom Inline Policy | Custom Inline Policy | Specific write permissions for operational tasks  |

## Inline Policy JSON

Save this as limited-working-access.json or paste directly into the IAM Identity Center inline policy editor:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EKSOperationsAccess",
      "Effect": "Allow",
      "Action": [
        "eks:AccessKubernetesApi"
      ],
      "Resource": "arn:aws:eks:*:*:cluster/cicd-*"
    },
    {
      "Sid": "CloudWatchLogsAccess",
      "Effect": "Allow",
      "Action": [
        "logs:GetLogEvents",
        "logs:FilterLogEvents",
        "logs:StartQuery",
        "logs:GetQueryResults",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3OperationsAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::cicd-artifacts-*",
        "arn:aws:s3:::cicd-artifacts-*/*"
      ]
    },
    {
      "Sid": "SSMStartSessionOnTaggedInstances",
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringLike": {
          "ssm:resourceTag/Environment": ["cicd-*"]
        }
      }
    },
    {
      "Sid": "SSMSessionDocuments",
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": [
        "arn:aws:ssm:*:*:document/SSM-SessionManagerRunShell",
        "arn:aws:ssm:*:*:document/AWS-StartInteractiveCommand",
        "arn:aws:ssm:*:*:document/AWS-StartPortForwardingSession"
      ]
    },
    {
      "Sid": "SSMManageOwnSessions",
      "Effect": "Allow",
      "Action": [
        "ssm:TerminateSession",
        "ssm:ResumeSession"
      ],
      "Resource": "arn:aws:ssm:*:*:session/${aws:username}-*"
    }
  ]
}
```

## Policy Breakdown

### EKS Operations Access

- Actions: DescribeCluster, ListClusters, ListNodegroups, DescribeNodegroup, AccessKubernetesApi
- Resource: Only clusters matching cicd-\* naming pattern
- Why: Engineers need to view cluster status and access the Kubernetes API through the AWS Console and CLI

### CloudWatch Logs Access

- Actions: GetLogEvents, FilterLogEvents, StartQuery, GetQueryResults, DescribeLogGroups
- Resource: All log groups (needed for cross-service troubleshooting)
- Why: Engineers need to read application and infrastructure logs for debugging

### S3 Operations Access

- Actions: GetObject, ListBucket, PutObject
- Resource: Only buckets matching cicd-artifacts-\*
- Why: Engineers need to read and upload build artifacts

### EC2 Read Access

- Actions: DescribeInstances, DescribeSecurityGroups, DescribeSubnets, DescribeVpcs
- Resource: All (read-only, no risk)
- Why: Engineers need to view infrastructure status for troubleshooting

### SSM Session Access

- Actions: StartSession, TerminateSession, ResumeSession
- Resource: EC2 instances tagged with Environment: cicd-\*
- Why: Engineers need to connect to CICD infrastructure instances for maintenance

## Before vs. After

| **Aspect**         | **Before (Shared IAM Key)**   | **After (IAM Identity Center)**          |
|--------------------|-------------------------------|------------------------------------------|
| **Permissions**    | Full admin-level              | Scoped to specific actions and resources |
| **Accountability** | None — shared key             | Individual user identity tracked         |
| **Credentials**    | Long-term, on Confluence      | Temporary, via SSO (1-12 hours)          |
| **Audit**          | CloudTrail shows service user | CloudTrail shows individual user         |
| **Rotation**       | Manual, risky                 | Automatic (temporary credentials)        |

## How to Apply the Permission Set

### Step 1: Create the Permission Set in IAM Identity Center

1. Navigate to IAM Identity Center → Permission sets
2. Click Create permission set
3. Choose Custom permission set
4. Name: LimitedWorkingAccess
5. Session duration: 4 hours (adjust based on your needs)
6. Add AWS managed policy: ReadOnlyAccess
7. Add Inline policy: paste the JSON from the [Inline Policy JSON](#inline-policy-json) section above
8. Review and create

### Step 2: Associate with AWS Accounts

1. Navigate to IAM Identity Center → AWS accounts
2. Select the target account(s)
3. Click Assign users or groups
4. Select the Entra ID group for operations engineers
5. Select the LimitedWorkingAccess permission set
6. Confirm assignment

## Customization

Adjust the inline policy based on your specific needs:

- Add services: If engineers need access to additional AWS services
- Restrict resources: Tighten resource ARNs to specific clusters, buckets, or instances
- Add conditions: Use aws:RequestedRegion to limit to specific regions
- Session duration: Balance between convenience (longer) and security (shorter)
