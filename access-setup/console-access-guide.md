# 🖥️ AWS Console Access Guide for Operations Engineers

## How to Access AWS Console via IAM Identity Center

### Step 1: Open the SSO Portal

Navigate to your organization's IAM Identity Center portal URL:

```yaml
https://myorg.awsapps.com/start
```

Replace myorg with your organization's actual SSO portal name.

You will be redirected to your corporate identity provider (Entra ID) for authentication.

### Step 2: Select Your AWS Account

After logging in, you'll see a list of AWS accounts you have access to.

1. Find the target account (e.g., CICD-Production - 123456789012)
2. Expand the account to see available roles
3. You should see: LimitedWorkingAccess

### Step 3: Launch the Management Console

Click Management Console next to the LimitedWorkingAccess role.

This opens the AWS Management Console with temporary credentials scoped to the permissions defined in the permission set.

### Step 4: Navigate to Amazon EKS

1. In the Console search bar, type EKS and select Amazon Elastic Kubernetes Service
2. Click Clusters in the left navigation
3. Select your CICD cluster (e.g., cicd-production)

### Step 5: View Cluster Resources

In the EKS Console, you can:

- Resources tab → View pods, deployments, services, configmaps
- Workloads → See running deployments and their status
- Service → View exposed services and endpoints
- Config and Storage → View configmaps and persistent volume claims
- View nodegroups and nodes

## Permissions Summary

### What You CAN Do

| **Action**                                  | **Status** |
|---------------------------------------------|------------|
| View pods and their status                  | Allowed    |
| View pod logs                               | Allowed    |
| View deployments, services, configmaps      | Allowed    |
| View CloudWatch logs                        | Allowed    |
| View EC2 instances                          | Allowed    |
| View S3 bucket contents (cicd-artifacts-\*) | Allowed    |
| Start SSM sessions to EC2 instances with the tag Environment=cicd-production        | Allowed    |

### What You CANNOT Do

| **Action**                             | **Status** |
|----------------------------------------|------------|
| Delete any resources                   | Denied     |
| Modify IAM roles or policies           | Denied     |
| Create or terminate EC2 instances      | Denied     |
| Access billing or cost information     | Denied     |
| Modify security groups or VPC settings | Denied     |
| Start SSM sessions to EC2 instances without the tag Environment=cicd-production        | Denied    |

## Session Duration

Your Console session lasts 4 hours. After expiration:

1. You'll see "Session expired" errors
2. Return to the SSO portal and click Management Console again
3. A new session with fresh temporary credentials will be created

## Troubleshooting

### "You are not authorized to perform this operation"

- This is expected for actions outside your permission set
- If you need additional permissions, contact the security team

### Can't see Kubernetes resources in EKS Console

- Verify you're logged in with the LimitedWorkingAccess role (check top-right corner)
- The EKS access entry must be configured for your role (contact the security team)

### Session expired during work

- Return to the SSO portal: https://myorg.awsapps.com/start
- Click Management Console again for a fresh session
