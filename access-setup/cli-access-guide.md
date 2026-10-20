# ⌨️ AWS CLI SSO Configuration Guide

## Prerequisites

- AWS CLI v2 installed (https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- kubectl installed (https://kubernetes.io/docs/tasks/tools/)
- Access to the IAM Identity Center portal

## Step 1: Configure AWS CLI with SSO

Run the interactive SSO configuration:

```bash
aws configure sso
```

When prompted, enter:

```
SSO session name (Recommended):  my-sso
SSO start URL [None]:            https://myorg.awsapps.com/start
SSO region [None]:               eu-west-1
SSO registration scopes [sso:account:access]:  sso:account:access
```

A browser window will open for authentication. Log in with your corporate credentials.

After authentication, the CLI will show available accounts and roles:

```
There are N AWS accounts available to you.
> CICD-Production, 123456789012
  CICD-PreProd, 987654321098
 
Using the account ID 123456789012
There are N roles available to you.
> LimitedWorkingAccess
 
CLI default client Region [None]: eu-west-1
CLI default output format [None]: json
CLI profile name [LimitedWorkingAccess-123456789012]: cicd-ops
```

## Step 2: Verify the Configuration

Check that the profile was created:

```bash
cat ~/.aws/config
```

Expected output:

```json
[profile cicd-ops]
sso_session = my-sso
sso_account_id = 123456789012
sso_role_name = LimitedWorkingAccess
region = eu-west-1
output = json
 
[sso-session my-sso]
sso_start_url = https://myorg.awsapps.com/start
sso_region = eu-west-1
sso_registration_scopes = sso:account:access
```

## Step 3: Log In via SSO

```bash
aws sso login --profile cicd-ops
```

A browser window opens for authentication. After successful login:

```
Successfully logged into Start URL: https://myorg.awsapps.com/start
```

## Step 4: Verify AWS Access

```bash
aws sts get-caller-identity --profile cicd-ops
```

Expected output:

```json
{
    "UserId": "AROA1234EXAMPLE:karolina@company.com",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_LimitedWorkingAccess_abc123/karolina@company.com"
}
```

## Step 5: Configure kubectl for EKS

```bash
aws eks update-kubeconfig \
    --name cicd-production \
    --region eu-west-1 \
    --profile cicd-ops
```

Expected output:

```
Added new context arn:aws:eks:eu-west-1:123456789012:cluster/cicd-production to /home/user/.kube/config
```

## Step 6: Verify Kubernetes Access

```bash
# List pods in the CICD namespace
kubectl get pods -n cicd-system
```

Expected output:

```
NAME                          READY   STATUS    RESTARTS   AGE
pipeline-runner-abc123        1/1     Running   0          2d
identity-mgr-def456          1/1     Running   0          5d
image-builder-ghi789         1/1     Running   0          1d
```

Test permissions:

```bash
# Should work - list pods
kubectl get pods -A
 
# Should work - view logs (rename the pod accordingly)
kubectl logs -n cicd-apps hello-web-abc123
 
# Should FAIL
kubectl exec -it cicd-apps hello-web-abc123 -- /bin/sh
 
# Should FAIL - delete pod
kubectl delete pod -n cicd-apps hello-web-abc123

```

## Daily Workflow

```bash
# 1. Login (once per session, lasts several hours)
aws sso login --profile cicd-ops
 
# 2. Use AWS CLI normally
aws eks list-clusters --profile cicd-ops
aws logs describe-log-groups --profile cicd-ops
 
# 3. Use kubectl normally
kubectl get pods -n cicd-apps
kubectl logs -n cicd-apps <pod-name>
```

## Credential Expiration

Temporary credentials expire after the configured session duration (default: 4 hours).

When credentials expire:

```bash
# You'll see errors like:
# "The SSO session associated with this profile has expired"
 
# Simply re-login:
aws sso login --profile cicd-ops
```

## Multiple Accounts

If you have access to multiple accounts (e.g., pre-prod and production), add profiles to ~/.aws/config:

```json
[profile cicd-preprod]
sso_session = my-sso
sso_account_id = 987654321098
sso_role_name = LimitedWorkingAccess
region = eu-west-1
output = json
```

### Register each cluster once (with a human-readable name)

Run `update-kubeconfig` once per cluster, using `--alias` to give each context a
short, memorable name instead of the long default
(`arn:aws:eks:eu-west-1:123456789012:cluster/cicd-production`):

```bash
aws eks update-kubeconfig --name cicd-preprod \
    --region eu-west-1 --profile cicd-preprod --alias cicd-preprod

aws eks update-kubeconfig --name cicd-production \
    --region eu-west-1 --profile cicd-ops --alias cicd-production
```

### Switch between clusters with kubectl

Once both contexts exist, switching is a kubectl operation — no need to re-run
`update-kubeconfig`:

```bash
# See the contexts you have (the * marks the current one)
kubectl config get-contexts

# Switch to pre-prod
kubectl config use-context cicd-preprod
kubectl get pods -n cicd-apps

# Switch to production
kubectl config use-context cicd-production
kubectl get pods -n cicd-apps
```

> 💡 Run a one-off command against a specific cluster without switching the
> current context by passing `--context`:
> ```bash
> kubectl --context preprod get pods -n cicd-apps
> ```

> ℹ️ The context only selects which cluster kubectl talks to. The matching AWS
> profile still provides the credentials, so make sure you've run
> `aws sso login --profile <profile>` for the account behind that cluster.

## Key Benefits

| **Benefit**              | **Description**                                            |
|--------------------------|------------------------------------------------------------|
| No long-term credentials | Nothing stored on disk that can be stolen                  |
| Auto-expiring            | Credentials expire automatically (1-12 hours configurable) |
| Individual identity      | Every API call traced in CloudTrail                 |
| Browser-based login      | Seamless SSO flow through the corporate identity provider |
| Profile-based            | Easy to switch between accounts and environments           |
