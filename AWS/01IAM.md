# AWS IAM (Identity and Access Management)

IAM is AWS's centralized service for controlling **who** can access your AWS account and **what** they can do. It manages authentication (who you are) and authorization (what you're allowed to do).

---

## Core Concepts

### Root User
- Created automatically when the AWS account is first set up.
- Has **unrestricted access** to all services and billing.
- **Best Practice**: Never use root for day-to-day tasks. Create an IAM admin user instead, and enable MFA on root.

### IAM Users
- Represent individual identities (people or applications).
- Can have **console access** (username + password) and/or **programmatic access** (access key + secret).
- Permissions can be assigned directly or through groups.

### IAM Groups
- A collection of IAM users. Assign permissions to the group; all members inherit them.
- Users can belong to **multiple groups**.
- Groups **cannot be nested** (no group inside a group).

### IAM Roles
- An identity with permissions, like a user — but **not tied to a specific person**.
- Assumed **temporarily** by trusted entities: AWS services, users, or external identities.
- No long-term credentials — generates temporary security tokens.

Common use cases:
- Let an **EC2 instance** access S3 without hardcoding credentials
- Allow a **Lambda function** to read from DynamoDB
- Grant **cross-account access** between AWS accounts
- Enable **federated users** (e.g., from Google or Active Directory) to use AWS

### IAM Policies
- JSON documents that define permissions.
- Can be attached to users, groups, or roles.

**Types:**
- **Identity-based policies**: Attached to users, groups, or roles.
- **Resource-based policies**: Attached directly to AWS resources (e.g., S3 bucket policies).
- **Permission Boundaries**: Limit the maximum permissions an identity-based policy can grant.

---

## Console Access vs Programmatic Access

| Access Type | Credentials Needed |
|-------------|-------------------|
| AWS Console | Account ID or alias, IAM username, Password |
| CLI / SDK / API | Access Key ID + Secret Access Key + Region |

> **Note**: Access keys are generated in the AWS Console. Save them immediately — the Secret Access Key is only shown once. Store them securely (never commit to git).

---

## IAM Policy Example

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```

This grants full access to all S3 resources. Common `Effect` values are `Allow` and `Deny`.

### More Granular Example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

---

## IAM Role — Trust Policy

A Role has two parts:

**Trust Policy** — who can assume this role:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**Permissions Policy** — what the role can do (same format as above).

---

## Useful CLI Commands

```bash
# List users and groups
aws iam list-users
aws iam list-groups

# Create a user
aws iam create-user --user-name alice

# Attach a managed policy to a user
aws iam attach-user-policy \
  --user-name alice \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# List attached policies for a user
aws iam list-attached-user-policies --user-name alice

# Get the current IAM identity (who am I?)
aws sts get-caller-identity
```

---

## IAM Best Practices

- **Enable MFA** on the root account and all privileged users.
- **Never use root** for everyday tasks.
- **Least privilege**: Grant only the permissions needed.
- **Use roles over access keys** for EC2, Lambda, and other AWS services.
- **Rotate access keys** regularly for users that require them.
- **Use groups** to manage permissions at scale — attach policies to groups, not individual users.
- **Use Permission Boundaries** to delegate admin tasks safely.
- **Audit with IAM Access Analyzer** to find overly permissive policies and unintended access.

---

## Quick Reference: ARN Format

Amazon Resource Names (ARNs) uniquely identify AWS resources:
```
arn:aws:iam::123456789012:user/alice
arn:aws:iam::123456789012:role/MyRole
arn:aws:s3:::my-bucket
```