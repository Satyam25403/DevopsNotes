# AWS CloudShell

AWS CloudShell is a browser-based shell environment pre-authenticated with your AWS Console credentials. It gives you instant access to the AWS CLI, scripting tools, and your account's resources — without installing anything locally.

---

## What Is AWS CloudShell?

- **Browser-based**: Launch directly from the AWS Management Console (icon in the top navigation bar).
- **Pre-authenticated**: Uses your Console session credentials — no `aws configure` needed.
- **Amazon Linux 2023**: Comes with AWS CLI v2, Python 3, Node.js, Git, pip, jq, and more preinstalled.
- **1 GB persistent storage per region**: Scripts, files, and configs survive between sessions.
- **Supports Bash, PowerShell, and Zsh**.

---

## How to Launch

1. Log into the [AWS Console](https://console.aws.amazon.com).
2. Click the **CloudShell icon** (terminal icon) in the top navigation bar, or search for "CloudShell" in the services search.
3. A terminal opens in the browser — you're ready to run AWS CLI commands immediately.

---

## Key Features

- **No setup**: No local install required — works from any device with a browser.
- **IAM-scoped**: All commands run with the permissions of your currently logged-in IAM user/role.
- **File upload/download**: Use the Actions menu to upload files into CloudShell or download files back to your machine.
- **Multiple tabs**: Open multiple CloudShell sessions simultaneously.
- **Safe for quick ops**: Great for one-off tasks, debugging, and exploration without risking your local environment.

---

## Common Use Cases

```bash
# List S3 buckets
aws s3 ls

# Check EC2 instances
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]" --output table

# Check who you're logged in as
aws sts get-caller-identity

# Get a parameter from SSM
aws ssm get-parameter --name "/myapp/prod/db-url" --with-decryption

# Invoke a Lambda function
aws lambda invoke --function-name myFunction output.json && cat output.json

# Check CloudWatch logs
aws logs tail /aws/lambda/myFunction --follow

# Edit a file with vim
vim my-script.sh
```

---

## Installing Additional Tools

Since CloudShell runs Amazon Linux 2023, you can install packages:
```bash
# Python packages
pip install boto3 pandas

# Node packages
npm install -g serverless

# Other tools
sudo dnf install -y jq
```

> **Note**: Installed packages do **not** persist between sessions (unlike files in your home directory). You'll need to reinstall them each session, or add install commands to a startup script.

---

## Persisting Work

Files in your home directory (`~`) persist across sessions (1 GB per region). Use this to store:
- Shell scripts you run often
- Configuration files
- Downloaded outputs

```bash
# Save a useful script
vim ~/describe-all.sh
chmod +x ~/describe-all.sh
```

---

## Limitations

- **No inbound network access**: You can't expose ports or run a web server.
- **Session timeout**: Sessions time out after inactivity (~20 minutes).
- **Packages don't persist**: Only files in the home directory persist.
- **Not for long-running processes**: Use EC2 + `nohup` or AWS services for that.
- **Region-specific**: Storage is per-region — files in `us-east-1` aren't visible in `us-west-2`.

---

## CloudShell vs Local CLI

| | CloudShell | Local AWS CLI |
|--|------------|---------------|
| Setup required | None | Yes (install + configure) |
| Authentication | Auto (Console session) | Access keys or SSO |
| Persistence | 1 GB home dir | Full local filesystem |
| Best for | Quick ops, exploration | Scripting, automation, CI/CD |