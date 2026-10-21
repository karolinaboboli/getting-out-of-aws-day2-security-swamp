# 🔄 IAM Access Key Rotation (Demo / Lab)

This is the **REMEDIATION** half of the IAM key demo: the zero-downtime rotation
that retires the long-term static key created in
[iam-keys-setup.md](iam-keys-setup.md) — create new key → switch → deactivate old
→ delete old.

> ⚠️ **Run this only after the alternative access exists and the team is trained.**
> In the walkthrough, rotation is the *last* step: the IAM Identity Center
> permission set, Kubernetes RBAC, and EKS access entry must already be in place,
> and operations engineers must be comfortable working via SSO, before the old
> key is removed. Removing it earlier would block them.

## Prerequisites

- You completed [iam-keys-setup.md](iam-keys-setup.md) and still have these set:

```bash
export AWS_REGION=eu-west-1
export IAM_USER=cicd-legacy-user
export DEMO_PROFILE=cicd-legacy
export ADMIN_POLICY_ARN=arn:aws:iam::aws:policy/AdministratorAccess
export OLD_KEY_ID=...     # the key id from setup; re-derive below if unset
```

If `$OLD_KEY_ID` is no longer set, look it up:

```bash
export OLD_KEY_ID=$(aws iam list-access-keys --user-name "$IAM_USER" \
    --query 'AccessKeyMetadata[0].AccessKeyId' --output text)
echo "Old key: $OLD_KEY_ID"
```

---

## 1. Create a Second (New) Access Key

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

## 2. Deactivate the Old Key

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
>     --query 'AccessKeyLastUsed.{when:LastUsedDate,service:ServiceName,region:Region}'
> ```
> Run the CloudTrail daily-usage query (`../cloudtrail-analysis/`) and wait for a
> clean period with zero calls on `$OLD_KEY_ID`.

---

## 3. Delete the Old Key

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
| Start (from setup)        | Active   | —        | —   |
| Create new key (Step 1)   | Active   | Active   | yes |
| Switch clients to new key | Active   | Active   | yes |
| Deactivate old (Step 2)   | Inactive | Active   | yes (set back to Active) |
| Delete old (Step 3)       | Deleted  | Active   | **no** |

The key safety property: there's always at least one working key, and the
destructive delete happens only after a reversible deactivation proves the old
key is no longer in use.
