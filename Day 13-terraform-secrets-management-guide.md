# Terraform Secrets Management — Advanced Reference Guide

A practical reference for handling sensitive data securely in Terraform across AWS environments.

---

## The Three Leak Paths and How to Close Each One

### 1. Hardcoded values in .tf files

Vulnerable:
```terraform
resource "aws_db_instance" "example" {
  username = "admin"
  password = "hardcoded-password"
}
```

Fix — use Secrets Manager data source. Secrets are fetched at runtime and never exist in your code.

### 2. Variable defaults

Vulnerable:
```terraform
variable "db_password" {
  default = "hardcoded-default"
}
```

Fix — no defaults for secrets. Use `sensitive = true` and provide values via environment variables or Secrets Manager.

### 3. State file plaintext storage

No code-level fix exists. Mitigation is backend security: S3 with `encrypt = true`, restricted IAM, versioning enabled, no public access.

---

## AWS Secrets Manager Integration Pattern

### Bootstrap (manual, one time)

```bash
aws secretsmanager create-secret \
  --region us-east-1 \
  --name "prod/db/credentials" \
  --secret-string file://credentials.json
```

`credentials.json`:
```json
{
  "username": "dbadmin",
  "password": "your-secure-password"
}
```

### Terraform data source pattern

```terraform
data "aws_secretsmanager_secret" "db" {
  name = "prod/db/credentials"
}

data "aws_secretsmanager_secret_version" "db" {
  secret_id = data.aws_secretsmanager_secret.db.id
}

locals {
  db_creds = jsondecode(
    data.aws_secretsmanager_secret_version.db.secret_string
  )
}
```

Usage:
```terraform
username = local.db_creds["username"]
password = local.db_creds["password"]
```

---

## HashiCorp Vault Integration Pattern

Use Vault when you need multi-cloud secret management, dynamic credentials, or fine-grained access policies beyond what AWS IAM provides.

```terraform
provider "vault" {
  address = "https://vault.example.com"
}

data "vault_generic_secret" "db" {
  path = "secret/prod/db"
}

resource "aws_db_instance" "example" {
  username = data.vault_generic_secret.db.data["username"]
  password = data.vault_generic_secret.db.data["password"]
}
```

When to use Vault vs Secrets Manager:

| Requirement | AWS Secrets Manager | HashiCorp Vault |
|---|---|---|
| AWS-only workloads | ✅ | ✅ |
| Multi-cloud | ❌ | ✅ |
| Dynamic credentials | Limited | ✅ |
| Native AWS IAM integration | ✅ | Partial |
| Self-hosted | ❌ | ✅ |

---

## Environment Variables for Provider Credentials

Never put AWS credentials in Terraform configuration files.

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

In CI/CD, inject from GitHub Actions secrets or use IAM roles for EC2/ECS/Lambda execution environments. IAM roles are always preferred over access keys for workloads running inside AWS.

---

## State File Security Checklist

```terraform
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}
```

Required controls:

- `encrypt = true` — AES-256 server-side encryption at rest
- S3 bucket versioning — recovery from accidental overwrites or corruption
- Block all public access — S3 bucket policy and account-level block
- DynamoDB locking — prevents concurrent state modifications
- IAM least-privilege — only Terraform execution roles can read/write state

---

## IAM Policy for Least-Privilege State Bucket Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::terraform-state-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::terraform-state-bucket"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/terraform-state-locks"
    }
  ]
}
```

---

## .gitignore Template for Terraform Projects

```
# Terraform state — contains secrets in plaintext
*.tfstate
*.tfstate.backup

# Terraform working directory — local cache, not source code
.terraform/

# Lock file — commit this, it pins provider versions
# !.terraform.lock.hcl

# Variable files — may contain environment-specific secrets
*.tfvars
*.tfvars.json

# Override files — local developer overrides, not for sharing
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Crash logs
crash.log
crash.*.log

# Windows metadata files from file copies
*:Zone.Identifier
```

---

## Common Issues and Fixes

**jsondecode failure** — caused by invalid JSON in the secret, usually from special characters in passwords when using inline `--secret-string`. Fix: use `file://credentials.json` instead.

**Stale state lock** — caused by interrupted apply leaving a DynamoDB lock record. Fix: `terraform force-unlock <lock-id>`. Only use when certain no other operation is running.

**Region mismatch on S3 bucket** — S3 buckets are region-specific and cannot be moved. Fix: create a new bucket in the target region or keep the backend in its original region (backend and provider regions can differ).

**Terraform prompting for removed variables** — variable declarations with no defaults still exist in variables.tf after switching to Secrets Manager. Fix: delete the variable blocks entirely and verify no `var.` references remain.

---

*Part of the 30-Day Terraform Challenge public learning journal.*

*GitHub: https://github.com/Mzokhulayo*
