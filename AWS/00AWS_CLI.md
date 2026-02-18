# AWS CLI (Command Line Interface)

AWS CLI is a powerful tool that lets you interact with AWS services directly from your terminal — perfect for developers, sysadmins, and cloud engineers who want to automate tasks, script deployments, or move faster than clicking through the console.

---

## What You Can Do with AWS CLI

- **Manage AWS resources**: EC2 instances, S3 buckets, IAM users, Lambda functions, and more
- **Automate workflows**: Use shell scripts or CI/CD pipelines to deploy infrastructure
- **Query and filter data**: Get JSON outputs and use JMESPath to extract exactly what you need
- **Switch profiles and regions**: Work across multiple AWS accounts with ease

---

## 1. Installation

### Linux
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Windows
```bash
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

### Verify Installation
```bash
aws --version
```

---

## 2. Configure Credentials

### Interactive Setup
```bash
aws configure
```
You'll be prompted for:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Output format (`json`, `text`, or `table`)

Credentials are stored at `~/.aws/credentials` and config at `~/.aws/config`.

### Using Environment Variables (recommended for CI/CD)
```bash
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_REGION=us-east-1
```

### Using Named Profiles (for multiple accounts)
```bash
aws configure --profile dev
aws configure --profile prod

# Use a specific profile
aws s3 ls --profile prod
```

Or set the default profile for your session:
```bash
export AWS_PROFILE=prod
```

---

## 3. Common Commands

```bash
# S3
aws s3 ls                                         # List all buckets
aws s3 ls s3://my-bucket/                         # List bucket contents
aws s3 cp myfile.txt s3://my-bucket/              # Upload file
aws s3 sync ./local-folder s3://my-bucket/        # Sync folder to S3

# EC2
aws ec2 describe-instances --region us-east-1
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# IAM
aws iam list-users
aws iam get-user --user-name alice

# Lambda
aws lambda invoke --function-name myFunction output.json
aws lambda list-functions

# SSM Parameter Store
aws ssm get-parameter --name "/myapp/db-url" --with-decryption
```

---

## 4. Output Formatting & Querying

```bash
# Change output format
aws ec2 describe-instances --output table
aws ec2 describe-instances --output text

# Use JMESPath to filter output
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].PublicIpAddress" \
  --output text
```

---

## 5. AWS CLI in CI/CD Pipelines

In GitHub Actions, set AWS credentials as secrets and use the official action:
```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1
```

---

## 6. Helpful Tips

```bash
# Get help for any service or command
aws help
aws ec2 help
aws s3 cp help

# Dry run (check permissions without executing)
aws ec2 run-instances --dry-run ...

# Use --no-cli-pager to avoid paging output in scripts
aws ec2 describe-instances --no-cli-pager
```

---

## 7. IAM Roles vs Access Keys

| Method | When to Use |
|--------|-------------|
| Access Keys | Local dev, CI/CD with secrets manager |
| IAM Role (on EC2/Lambda) | Always preferred for AWS-hosted workloads — no hardcoded credentials |

> **Best Practice**: Never hardcode credentials in your code. Use IAM roles for EC2/Lambda and environment variables or AWS Secrets Manager for local dev.