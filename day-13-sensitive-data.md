# How to Handle Sensitive Data Securely in Terraform

*Day 13 of the 30-day IaC challenge — three secret leak paths, AWS Secrets Manager integration, sensitive value handling, and state file security*

---

Every real infrastructure deployment involves secrets. The number one security mistake Terraform engineers make is letting those secrets end up where they should never be — hardcoded in `.tf` files, committed to Git, or stored in plaintext in state files. This post covers every leak path and how to close each one.

---

## The Three Secret Leak Paths

### Leak Path 1 — Hardcoded in .tf files

```terraform
# Never do this
resource "aws_db_instance" "example" {
  username = "admin"
  password = "super-secret-password"
}
```

The moment you run `git add`, this is in your commit history permanently. Even if you delete it later, it exists forever in Git history. Anyone with repo access — now or in the future — can recover it.

Secure alternative — fetch from Secrets Manager at runtime:

```terraform
data "aws_secretsmanager_secret" "db_credentials" {
  name = "prod/db/credentials"
}

data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = data.aws_secretsmanager_secret.db_credentials.id
}

locals {
  db_credentials = jsondecode(
    data.aws_secretsmanager_secret_version.db_credentials.secret_string
  )
}

resource "aws_db_instance" "example" {
  username = local.db_credentials["username"]
  password = local.db_credentials["password"]
}
```

The secret is fetched at apply time from AWS. It never touches your `.tf` files.

---

### Leak Path 2 — Secrets as variable defaults

```terraform
# Also wrong
variable "db_password" {
  default = "super-secret-password"
}
```

Default values are stored in your `.tf` files and committed to source control. Secrets must never have defaults.

Secure alternative:

```terraform
variable "db_password" {
  description = "Database administrator password"
  type        = string
  sensitive   = true
  # No default — provide via TF_VAR_db_password or Secrets Manager
}
```

---

### Leak Path 3 — Secrets in the state file

Even if you handle the first two correctly, Terraform stores the values of sensitive resource attributes in `terraform.tfstate` in plaintext. Anyone with read access to the state file can see all your secrets.

This is why remote state with encryption and restricted access is non-negotiable:

```terraform
terraform {
  backend "s3" {
    bucket         = "terraform-state-mzokhulayo-us-east-1"
    key            = "stage/data-stores/mysql/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-up-and-running-locks"
    encrypt        = true
  }
}
```

---

## AWS Secrets Manager Integration

Create the secret manually — never through Terraform for bootstrap secrets:

```bash
aws secretsmanager create-secret \
  --region us-east-1 \
  --name "prod/db/credentials" \
  --secret-string file://secret.json
```

Where `secret.json` contains:

```json
{
  "username": "dbadmin",
  "password": "your-secure-password-here"
}
```

Using a file avoids shell escaping issues with special characters in passwords.

Fetch and use in Terraform:

```terraform
data "aws_secretsmanager_secret" "db_credentials" {
  name = "prod/db/credentials"
}

data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = data.aws_secretsmanager_secret.db_credentials.id
}

locals {
  db_credentials = jsondecode(
    data.aws_secretsmanager_secret_version.db_credentials.secret_string
  )
}

resource "aws_db_instance" "example" {
  engine              = "mysql"
  engine_version      = "8.0"
  instance_class      = "db.t3.micro"
  db_name             = "appdb"
  allocated_storage   = 10
  skip_final_snapshot = true

  username = local.db_credentials["username"]
  password = local.db_credentials["password"]
}
```

---

## Sensitive Variables and Outputs

```terraform
variable "db_password" {
  description = "Database administrator password"
  type        = string
  sensitive   = true
}

output "db_connection_string" {
  value     = "mysql://${aws_db_instance.example.username}@${aws_db_instance.example.endpoint}"
  sensitive = true
}
```

When a sensitive output is involved, Terraform shows this in plan and apply output:

```
+ db_connection_string = (sensitive value)
```

**Important:** `sensitive = true` does NOT prevent the value from being stored in state. It only prevents it from appearing in terminal output and logs. The real protection is your backend security.

---

## State File Security Checklist

- Remote backend with S3 and `encrypt = true`
- DynamoDB table for state locking
- S3 bucket versioning enabled
- Block all public access on the S3 bucket
- IAM policy restricting bucket access to only Terraform execution roles
- `.terraform/`, `*.tfstate`, `*.tfstate.backup`, and `*.tfvars` in `.gitignore`

Minimum IAM policy for state bucket access:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::terraform-state-mzokhulayo-us-east-1/*"
}
```

---

## .gitignore for Terraform Projects

```
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.backup
*.tfvars
override.tf
override.tf.json
```

Every entry exists for a reason. `.tfstate` files contain secrets in plaintext. `.tfvars` files often contain environment-specific values including credentials. `.terraform/` is a local cache that should never be committed.

---

## Key Learnings

`sensitive = true` hides values from plan output and logs. It does not prevent storage in state. The state file is the last line of defence — which is why backend encryption and access control are non-negotiable.

AWS Secrets Manager is the right choice for AWS workloads — native integration, IAM-based access control, and automatic rotation support. HashiCorp Vault is the right choice for multi-cloud environments or teams that need dynamic secrets, leasing, and fine-grained access policies across multiple platforms.

Some secrets cannot be removed from state entirely. Resources like `aws_db_instance` require username and password to manage the resource — Terraform must store these to compare desired state against actual state. The only mitigation is securing the state file itself.

---

## Challenges Hit

The `jsondecode` error caused by special characters in passwords — specifically a trailing `@"` sequence that broke JSON parsing. Fixed by using `file://secret.json` instead of inline `--secret-string` to avoid shell escaping issues entirely.

Stale DynamoDB state lock from a previous interrupted apply — fixed with `terraform force-unlock`.

Region mismatch between existing S3 bucket and provider region — resolved by creating a new bucket in the target region rather than trying to move the existing one.

---

*Next post: Terraform testing — how to validate your infrastructure before it hits production.*

*@HashiCorp User Group Meru | @AWS AI/ML User Group Kenya | @EveOps*

*#Terraform #AWS #InfrastructureAsCode #Security #DevOps #30DayTerraformChallenge*
