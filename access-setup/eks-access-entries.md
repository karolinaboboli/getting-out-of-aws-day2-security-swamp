# 🔗 EKS Access Entries Configuration Guide

## Overview

EKS access entries connect AWS IAM identities to Kubernetes RBAC. This allows IAM Identity Center users to access the EKS cluster with the permissions defined in the Kubernetes ClusterRole.

## Architecture

```
IAM Identity Center Role
        │
        ▼
EKS Access Entry (AWS side)
   maps IAM role → K8s group "ops-engineers"
        │
        ▼
ClusterRoleBinding (K8s side)
   maps group "ops-engineers" → ClusterRole "ops-limited-access"
        │
        ▼
ClusterRole (K8s side)
   defines: get, list, watch, logs, exec
```

## Prerequisites

- EKS cluster with access entries enabled (API authentication mode)
- The IAM Identity Center role ARN for the LimitedWorkingAccess permission set
- Kubernetes RBAC resources applied (see [Kubernetes RBAC configuration](../kubernetes/README.md))

## Find the IAM Role ARN

The IAM Identity Center creates a role in each account. Find it:

### AWS Console

1. Navigate to IAM → Roles
2. Search for AWSReservedSSO_LimitedWorkingAccess
3. Copy the full ARN

### AWS CLI

```bash
aws iam list-roles \
    --query "Roles[?contains(RoleName, 'LimitedWorkingAccess')].Arn" \
    --output text
```

Expected output:

```yaml
arn:aws:iam::123456789012:role/aws-reserved/sso.amazonaws.com/eu-west-1/AWSReservedSSO_LimitedWorkingAccess_abc123def456
```

## Create the EKS Access Entry

### AWS Console

1. Navigate to Amazon EKS → Clusters → select your cluster
2. Go to the Access tab
3. Click Create access entry
4. Configure:

| **Setting**       | **Value**                                                                                                                |
|-------------------|--------------------------------------------------------------------------------------------------------------------------|
| IAM principal ARN | arn:aws:iam::123456789012:role/aws-reserved/sso.amazonaws.com/eu-west-1/AWSReservedSSO_LimitedWorkingAccess_abc123def456 |
| Type              | STANDARD                                                                                                                 |
| Kubernetes groups | ops-engineers                                                                                                            |

1. Click Create

> *Important: The Kubernetes group name "ops-engineers" must match the subjects.name in the ClusterRoleBinding YAML.*

### AWS CLI

```bash
# get role ARN

export LIMITED_WORKING_ACCESS_ROLE_ARN=$(aws iam list-roles \
    --query "Roles[?contains(RoleName, 'LimitedWorkingAccess')].Arn" \
    --output text)

export CLUSTER_NAME=cicd-production
export AWS_REGION=eu-west-1

# create access entry
aws eks create-access-entry \
    --cluster-name $CLUSTER_NAME \
    --principal-arn "$LIMITED_WORKING_ACCESS_ROLE_ARN" \
    --type STANDARD \
    --kubernetes-groups '["ops-engineers"]' \
    --region $AWS_REGION

# associate the AmazonEKSAdminView policy
aws eks associate-access-policy \
    --cluster-name "$CLUSTER_NAME" \
    --principal-arn "$LIMITED_WORKING_ACCESS_ROLE_ARN" \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSAdminViewPolicy \
    --access-scope type=cluster \
    --region "$AWS_REGION"
```

## Verify the Access Entry

```bash
# List all access entries
aws eks list-access-entries \
    --cluster-name $CLUSTER_NAME \
    --region $AWS_REGION
 
# Describe the specific entry
aws eks describe-access-entry \
    --cluster-name $CLUSTER_NAME \
    --principal-arn "$LIMITED_WORKING_ACCESS_ROLE_ARN" \
    --region $AWS_REGION
```

## Troubleshooting

### "AccessDeniedException" when creating access entry

- Verify the cluster's authentication mode includes API (not just CONFIG_MAP)
- Check: EKS → Cluster → Access tab → Authentication mode

### Users can't see Kubernetes resources in Console

- Verify the access entry exists and maps to the correct Kubernetes group
- Verify the ClusterRoleBinding exists in the cluster
- Check that the user is logged in with the correct permission set
