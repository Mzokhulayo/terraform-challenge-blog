# Preparing for the Terraform Associate Exam — Key Resources and Tips

*Day 23 of the 30-day IaC challenge — honest self-audit, CLI command mastery, and a structured study plan for the final days before certification.*

---

The book is finished. The pipeline is running. Now it is time to shift focus entirely toward the Terraform Associate certification exam. Today I audited myself against every official exam objective, identified exactly where my gaps are, and built a concrete study plan for the remaining days.

---

## The Official Exam Domains

The exam covers nine domains with different weightings. The CLI commands domain at 26% is the largest single section and the one most candidates underestimate. It is not enough to know what the commands do in general — the exam tests edge cases like what happens to real infrastructure when you run terraform state rm, or what terraform validate does that terraform plan does not.

The official study guide is the only authoritative source: https://developer.hashicorp.com/terraform/tutorials/certification-003/associate-study-003

---

## My Self-Audit

After 22 days of hands-on work, most domains are green. The two areas that need focused study are HCL functions and expressions, and Terraform Cloud capabilities in depth — specifically remote runs, workspace variable types, and how Sentinel fits into the workflow.

This kind of honest audit is more valuable than studying everything equally. Time spent on green areas is wasted. Time spent on red areas compounds.

---

## CLI Commands — What You Must Know Cold

The exam tests the CLI at a level most candidates do not expect. For each command you need to know what it does, what it does NOT do, and what happens to real infrastructure when you run it.

`terraform init` — initialises the working directory, downloads providers, and configures the backend. Running it again after a configuration change re-downloads providers and updates the lock file.

`terraform validate` — checks that the configuration is syntactically valid and internally consistent. It runs entirely locally with no AWS connection. It does not show what will change in real infrastructure.

`terraform fmt` — reformats code to canonical style. With `-recursive` it formats every file in the directory tree. The `-check` flag returns a non-zero exit code if files need formatting without modifying them — useful in CI.

`terraform plan` — connects to AWS, reads the current state, and shows exactly what will change if applied. This is the most important command in the workflow. Always save the output with `-out=plan.tfplan`.

`terraform apply` — carries out the changes shown in the plan. When passed a saved plan file it applies exactly what was reviewed. Without a plan file it generates a new plan first.

`terraform destroy` — tears down all resources managed by the current configuration. Equivalent to running apply with every resource marked for deletion.

`terraform output` — prints all output values from the current state. Useful in scripts to retrieve values like ALB DNS names or database endpoints.

`terraform state list` — lists all resources currently tracked in the state file. Does not show attributes, just resource addresses.

`terraform state show` — shows the full attributes of a specific resource as Terraform knows them. Usage: `terraform state show aws_instance.example`.

`terraform state mv` — moves a resource from one state address to another without destroying and recreating the real infrastructure. Used when renaming resources or moving them into modules.

`terraform state rm` — removes a resource from state without touching real infrastructure. The resource continues to exist in AWS but Terraform no longer manages it.

`terraform import` — brings existing infrastructure into Terraform state. You write the resource block first, then run import to associate the real resource with it.

`terraform workspace` — manages workspaces. Used with `new`, `select`, `list`, and `delete` subcommands to create and switch between isolated state environments.

`terraform login` — authenticates to Terraform Cloud by storing a token locally. Required before using HCP Terraform as a backend.

`terraform graph` — outputs a dependency graph in DOT format showing how resources depend on each other. Useful for debugging complex module relationships.

---

## Non-Cloud Providers

Terraform manages more than just cloud resources. The random and local providers appear frequently in exam questions.

```terraform
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

resource "random_password" "db_password" {
  length  = 16
  special = true
}

output "bucket_name" {
  value = "my-app-bucket-${random_id.bucket_suffix.hex}"
}

output "db_password" {
  value     = random_password.db_password.result
  sensitive = true
}
```

The random provider solves a real problem — S3 bucket names are globally unique across all AWS accounts, so appending a random suffix prevents naming conflicts when multiple teams or environments deploy the same configuration. The random_password resource generates secure passwords without hardcoding them in configuration files.

---

## Five Practice Questions

**Question 1:** You run `terraform state rm aws_s3_bucket.logs`. What happens to the real S3 bucket?

The bucket continues to exist in AWS. terraform state rm only removes the resource from Terraform's state file. It does not interact with AWS at all.

**Question 2:** A team member manually deleted an EC2 instance that Terraform manages. What happens on the next terraform plan?

Terraform detects the drift and shows the instance as a resource to be created. Terraform always checks real infrastructure against state during plan — it does not ignore changes made outside of Terraform.

**Question 3:** You have two AWS provider blocks — one default for us-east-1 and one with alias = "west" for us-west-2. A resource with no provider argument gets created in which region?

us-east-1. The provider without an alias is the default. Resources only use aliased providers when explicitly specified with the provider argument.

**Question 4:** What does terraform validate do that terraform plan does not?

Nothing additional — validate does less than plan. validate checks syntax locally without connecting to AWS. plan does everything validate does plus connects to AWS and shows what will change. The key distinction is that validate has no cloud dependency.

**Question 5:** You want to rename aws_instance.web to aws_instance.webserver without destroying the real EC2 instance. Which command?

terraform state mv aws_instance.web aws_instance.webserver. This updates the state address without touching real infrastructure. Running apply after renaming in the configuration without state mv would destroy the old instance and create a new one.

---

## Study Plan for Remaining Days

The two focus areas before the exam are HCL built-in functions — try to understand string, collection, and type conversion functions — and Terraform Cloud workflow details including the difference between remote and local runs, how workspace variables work, and where Sentinel fits in the plan/apply sequence.

The official practice questions are the best proxy for exam difficulty. Work through all of them. For every wrong answer, write down the correct answer in your own words before moving on.

---

*@HashiCorp User Group Meru | @AWS AI/ML User Group Kenya | @EveOps*

*#Terraform #TerraformAssociate #CertificationPrep #DevOps #30DayTerraformChallenge*
