# 🔑 IAM User & Access Key Lifecycle (Demo / Lab)

This walkthrough **recreates the incident** the talk is about — a long-term IAM
user with admin access and a static access key — and then demonstrates the
**zero-downtime key rotation** used to remediate it (create new key → switch →
deactivate old → delete old).

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

## 6. Create a Second (New) Access Key

An IAM user can have **two** access keys at once — this is what makes rotation
zero-downtime. Create the replacement while the old one is still working:

```bash
NEW_OUT=$(aws iam create-access-key --user-name "$IAM_USER")

export NEW_KEY_ID=$(echo "$NEW_OUT" | jq -r '.AccessKey.AccessKeyId')
export NEW_SECRET=$(echo "$NEW_OUT" | jq -r '.AccessKey.SecretAccessKey')

echo "New key: $NEW_KEY_ID"
```

Point the profile at the new key:

```bash
aws configure set aws_access_key_id     "$NEW_KEY_ID" --profile "$DEMO_PROFILE"
aws configure set aws_secret_access_key "$NEW_SECRET" --profile "$DEMO_PROFILE"

# Confirm the new key works before touching the old one
aws sts get-caller-identity --profile "$DEMO_PROFILE"
```

List both keys to confirm two now exist:

```bash
aws iam list-access-keys --user-name "$IAM_USER" \
    --query 'AccessKeyMetadata[].{id:AccessKeyId,status:Status,created:CreateDate}'
```

---

## 7. Deactivate the Old Key

Don't delete yet — **deactivate** first. This is reversible, so if something
still depends on the old key you can flip it back to `Active` without data loss.

```bash
aws iam update-access-key \
    --user-name "$IAM_USER" \
    --access-key-id "$OLD_KEY_ID" \
    --status Inactive
```

Verify the old key is `Inactive` and the new one is `Active`:

```bash
aws iam list-access-keys --user-name "$IAM_USER" \
    --query 'AccessKeyMetadata[].{id:AccessKeyId,status:Status}'
```

> 🔎 Before deleting, confirm the old key really is unused. Check its last-used
> timestamp and watch it stop advancing:
> ```bash
> aws iam get-access-key-last-used --access-key-id "$OLD_KEY_ID" \
    --query 'AccessKeyLastUsed.{when:LastUsedDate,service:ServiceName,region:Region}'
> ```
> Run the CloudTrail daily-usage query (`../cloudtrail-analysis/`) and wait for a
> clean period with zero calls on `$OLD_KEY_ID`.

---

## 8. Delete the Old Key

Once you're confident the old key is unused, delete it permanently:

```bash
aws iam delete-access-key \
    --user-name "$IAM_USER" \
    --access-key-id "$OLD_KEY_ID"
```

Confirm only the new key remains:

```bash
aws iam list-access-keys --user-name "$IAM_USER" \
    --query 'AccessKeyMetadata[].{id:AccessKeyId,status:Status}'
```

---

## Cleanup (tear down the whole demo)

```bash
# Delete remaining access key(s)
aws iam delete-access-key --user-name "$IAM_USER" --access-key-id "$NEW_KEY_ID"

# Detach the managed policy
aws iam detach-user-policy --user-name "$IAM_USER" --policy-arn "$ADMIN_POLICY_ARN"

# Delete the user
aws iam delete-user --user-name "$IAM_USER"

# Remove the local CLI profile entries
aws configure set aws_access_key_id     "" --profile "$DEMO_PROFILE"
aws configure set aws_secret_access_key "" --profile "$DEMO_PROFILE"
# (or edit ~/.aws/credentials and remove the [cicd-legacy] block)
```

---

## Rotation Summary

| Step | Old key state | New key state | Reversible? |
|------|---------------|---------------|-------------|
| Start                     | Active   | —        | —   |
| Create new key (Step 6)   | Active   | Active   | yes |
| Switch clients to new key | Active   | Active   | yes |
| Deactivate old (Step 7)   | Inactive | Active   | yes (set back to Active) |
| Delete old (Step 8)       | Deleted  | Active   | **no** |

The key safety property: there's always at least one working key, and the
destructive delete happens only after a reversible deactivation proves the old
key is no longer in use.
