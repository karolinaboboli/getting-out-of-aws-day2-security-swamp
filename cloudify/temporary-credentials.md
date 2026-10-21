# ☁️ Cloudify: Switching to Temporary Credentials

## Background

Cloudify was originally configured with a single long-term IAM access key ("the installation key"). As part of the security remediation, we switched to temporary credentials obtained via IAM Identity Center SSO.

## How It Works

1. Engineer authenticates via IAM Identity Center SSO
2. Obtains temporary credentials (Access Key ID + Secret Access Key + Session Token)
3. Places credentials on the EC2 instance running Cloudify
4. Cloudify uses these credentials for infrastructure operations
5. Credentials expire after the configured session duration

## Obtaining Temporary Credentials

### Option 1: From the SSO Portal

1. Log in to https://myorg.awsapps.com/start
2. Select the target account
3. Click Command line or programmatic access
4. Copy the credentials from Option 1 (environment variables) or Option 2 (aws credentials file)

### Option 2: From the AWS CLI

```bash
# Login first
aws sso login --profile cicd-ops
 
# Export credentials as environment variables
eval $(aws configure export-credentials --profile cicd-ops --format env)
 
# Or get them as JSON
aws configure export-credentials --profile cicd-ops --format process
```

## Placing Credentials on the Cloudify EC2 Instance

### Via SSM Session Manager

```bash
# Connect to the Cloudify EC2 instance
aws ssm start-session \
    --target i-0abc123def456789 \
    --profile cicd-ops
 
# On the EC2 instance, set the credentials
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
 
# Verify
aws sts get-caller-identity
```

### Via AWS Credentials File on EC2

```bash
# On the EC2 instance
cat > ~/.aws/credentials << EOF
[default]
aws_access_key_id = ASIA...
aws_secret_access_key = ...
aws_session_token = ...
EOF
```

## Accepted Risks and Trade-offs

### Risk 1: EC2 Session Hijacking

Scenario: Another user with SSH or SSM access to the Cloudify EC2 instance could:

1. List active sessions or processes
2. Read environment variables or the credentials file
3. Use another engineer's temporary credentials

Mitigation:

- Temporary credentials expire (unlike the old long-term key)
- CloudTrail logs show the original identity, aiding investigation
- SSM session logging provides audit trail of who connected

Decision: Accepted as lesser risk than shared long-term credentials on Confluence.

### Risk 2: Credential Expiration During Long Operations

Scenario: Cloudify upgrades can take several hours. If temporary credentials expire mid-operation:

- The upgrade fails partway through
- Manual intervention required to resume
- Potential for inconsistent state

Mitigation:

- Use maximum session duration (12 hours) for Cloudify operations
- Plan upgrades to start early in the day
- Document the recovery procedure for interrupted upgrades

> *This DID happen: We experienced interruptions during upgrades. It was problematic but manageable.*

Decision: Accepted as lesser risk than shared long-term credentials.

## Architects' Decision

> *Accept these risks as the lesser evil compared to shared long-term credentials published on Confluence. The temporary credentials approach provides:*
> *• Individual accountability (each engineer uses their own credentials)*
> *• Time-limited exposure (credentials expire automatically)*
> *• Audit trail (CloudTrail shows who did what)*
