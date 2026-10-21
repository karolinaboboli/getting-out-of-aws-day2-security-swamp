# 🔑 IAM User & Access Key — Initial Setup (Demo / Lab)

This is the **SETUP** half of the IAM key demo. It **recreates the incident** the
talk is about — a long-term IAM user with admin access and a static access key —
and gets that key actively making calls so there's something to analyze and later
rotate. The rotation itself lives in [iam-keys-rotation.md](iam-keys-rotation.md).

> ⚠️ **This is a lab demo, not a recommendation.** Long-term IAM user keys with
> `AdministratorAccess` are exactly the anti-pattern this project exists to get
> rid of. Run this only in a throwaway/sandbox account to generate CloudTrail
> data and practice rotation. In production, use IAM Identity Center + temporary
> credentials (see `../access-setup/`).

## Environment Variables

```bash
export AWS_REGION=eu-west-1
export IAM_USER=cicd-legacy-user          # the demo "installation key" user
export ADMIN_POLICY_ARN=arn:aws:iam::aws:policy/AdministratorAccess
export DEMO_PROFILE=cicd-legacy           # CLI profile that uses the static key
```

---

## 1. Create the IAM User

```bash
aws iam create-user \
    --user-name "$IAM_USER" \
    --tags Key=Purpose,Value=demo-rotation Key=Environment,Value=cicd-production
```

Verify:

```bash
aws iam get-user --user-name "$IAM_USER" --query 'User.{name:UserName,arn:Arn,created:CreateDate}'
```

---

## 2. Attach the AdministratorAccess Policy

```bash
aws iam attach-user-policy \
    --user-name "$IAM_USER" \
    --policy-arn "$ADMIN_POLICY_ARN"
```

Verify:

```bash
aws iam list-attached-user-policies --user-name "$IAM_USER"
```

---

## 3. Create the First Access Key

Capture the key id and secret into variables (the secret is shown **only once**):

```bash
CREATE_OUT=$(aws iam create-access-key --user-name "$IAM_USER")

export OLD_KEY_ID=$(echo "$CREATE_OUT" | jq -r '.AccessKey.AccessKeyId')
export OLD_SECRET=$(echo "$CREATE_OUT" | jq -r '.AccessKey.SecretAccessKey')

echo "Old key: $OLD_KEY_ID"
```

> 🔒 In real life this secret would never be echoed or stored. Here it only lives
> in a shell variable so the next step can configure a profile.

---

## 4. Configure the AWS CLI with the Key

Write a named profile backed by the static key (this is the "key on disk" the
talk warns about):

```bash
aws configure set aws_access_key_id     "$OLD_KEY_ID" --profile "$DEMO_PROFILE"
aws configure set aws_secret_access_key "$OLD_SECRET" --profile "$DEMO_PROFILE"
aws configure set region                "$AWS_REGION" --profile "$DEMO_PROFILE"
aws configure set output                json          --profile "$DEMO_PROFILE"
```

Confirm the profile authenticates as the new user:

```bash
aws sts get-caller-identity --profile "$DEMO_PROFILE"
# Arn should end in :user/cicd-legacy-user
```

> New access keys can take a few seconds to become active. If the call fails with
> `InvalidClientTokenId`, wait ~10s and retry.

---

## 5. Generate Some Activity (populate CloudTrail)

Make a handful of harmless read-only calls **using the static key**. These show
up in CloudTrail attributed to `$OLD_KEY_ID`, which is what the rotation analysis
later tracks.

```bash
aws sts get-caller-identity           --profile "$DEMO_PROFILE"
aws iam list-users                    --profile "$DEMO_PROFILE"
aws s3 ls                             --profile "$DEMO_PROFILE"
aws ec2 describe-regions              --profile "$DEMO_PROFILE" --region "$AWS_REGION"
aws ec2 describe-instances            --profile "$DEMO_PROFILE" --region "$AWS_REGION"
```

> ⏱️ CloudTrail management events typically appear within ~5–15 minutes. Use the
> queries in `../cloudtrail-analysis/` filtered on `userIdentity.accessKeyId =
> "$OLD_KEY_ID"` to see this activity.

---

## Next

The old key now exists and is in use. Start the CloudTrail analysis for it (see
`../cloudtrail-analysis/`), and once the alternative access (IAM Identity Center)
is in place and the team is trained, rotate the key away following
[iam-keys-rotation.md](iam-keys-rotation.md).

> 💡 Keep `$IAM_USER`, `$DEMO_PROFILE`, and `$OLD_KEY_ID` set in your shell — the
> rotation guide reuses them.
