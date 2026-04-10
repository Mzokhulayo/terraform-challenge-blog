#
A Workflow for Deploying Infrastructure Code with Terraform

*Day 21 of the 30-day IaC challenge — seven steps, infrastructure-specific safeguards, and Sentinel policies as the enforcement layer.*

---

Yesterday I mapped the seven-step application deployment workflow to Terraform. Today I ran a real infrastructure change through all seven steps and documented every place the infrastructure workflow needs something the application workflow does not.

---

## The Seven Steps

**Step 1 — Version control**

Terraform configuration in Git. Main branch protected — no direct pushes, changes only via pull request, status checks must pass before merge. State file in S3 with versioning enabled, never in the repository.

**Step 2 — Run locally**

Generated and saved the plan before making any change:

```bash
terraform plan -out=day21.tfplan
```

Plan showed 9 resources to add, 0 to change, 0 to destroy. New output variables exposed: `alb_dns_name` and `instance_security_group_id`. No destructions — no extra approval needed.

**Step 3 — Feature branch**

```bash
git checkout -b add-cloudwatch-alarms-day21
# Added instance_security_group_id output to outputs.tf
git commit -m "Add instance security group ID output for Day 21"
git push origin add-cloudwatch-alarms-day21
```

**Step 4 — Pull request**

PR #2 opened at github.com/Mzokhulayo/30-Days-Terraform-Challenge/pull/2 with plan output pasted as a comment. PR description included:

- What this changes: exposes instance security group ID as an output for consumers that need to add ingress rules externally
- Resources affected: 0 created, 0 modified, 0 destroyed — output-only change
- Blast radius: zero — outputs do not affect existing resources
- Rollback plan: revert the outputs.tf change and apply

**Step 5 — Automated tests**

GitHub Actions triggered on the PR. Unit tests passed in 15 seconds. Integration tests skipped on PR as configured.

**Step 6 — Merge and release**

```bash
git tag -a "v1.4.0" -m "Add instance security group ID output"
git push origin v1.4.0
```

**Step 7 — Deploy**

Applied from the saved plan file — not from a fresh plan:

```bash
terraform apply day21.tfplan
```

Ran plan immediately after to confirm clean state.

---

## Infrastructure-Specific Safeguards

**Plan file pinning**

Always apply from a saved plan file. The gap between `terraform plan` and `terraform apply` can introduce drift if infrastructure changes between the two runs. Someone else's apply, an auto-scaling event, or a console change can all cause the fresh plan to differ from what was reviewed.

```bash
# Correct
terraform plan -out=reviewed.tfplan
terraform apply reviewed.tfplan

# Risky
terraform apply
```

**Approval gates for destructive changes**

Any plan showing resource destruction requires explicit sign-off before apply. A plan that shows `0 to destroy` can go through normal PR review. A plan that shows `3 to destroy` needs a second pair of eyes specifically on those destructions.

**State backup verification**

S3 versioning confirmed enabled:

```bash
aws s3api get-bucket-versioning \
  --bucket terraform-state-mzokhulayo-us-east-1 \
  --region us-east-1
# { "Status": "Enabled" }
```

If an apply corrupts state, list available versions and restore:

```bash
aws s3api list-object-versions \
  --bucket terraform-state-mzokhulayo-us-east-1 \
  --prefix stage/data-stores/webserver-cluster/terraform.tfstate
```

**Blast radius documentation**

Every PR touching shared infrastructure must document dependencies. For this change the blast radius was zero — adding an output variable does not affect any existing resource. A change to a VPC CIDR block or a shared security group would have a much larger blast radius requiring explicit documentation of every dependent resource.

---

## Sentinel Policies

Sentinel is Terraform Cloud's policy-as-code framework. It runs after plan and before apply — enforcing rules that `terraform validate` cannot. Validate checks syntax. Sentinel checks intent.

```hcl
import "tfplan/v2" as tfplan

allowed_instance_types = ["t2.micro", "t2.small", "t2.medium", "t3.micro", "t3.small"]

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type is not "aws_instance" or
    rc.change.after.instance_type in allowed_instance_types
  }
}
```

This policy blocks any plan that would create an EC2 instance with an instance type outside the allowed list. A developer cannot bypass it by editing Terraform code — the policy runs server-side in Terraform Cloud before apply is permitted.

Sentinel can also enforce: mandatory tags on all resources, no production deployments outside business hours, required approval for any plan with destructions, and cost limits per workspace.

---

## Key Differences from Application Deployment

**Plan files have no application equivalent.** Application code is deterministic at build time. Infrastructure plans can change between generation and application if the real environment drifts. Saving the plan file is the only way to guarantee what was reviewed is what gets applied.

**Blast radius is unique to infrastructure.** A bad application deployment returns errors. A bad infrastructure deployment can destroy a production database. Every change needs a documented answer to "what breaks if this fails midway?"

**State is a shared mutable artifact.** Application deployments do not share state between team members. Terraform state is shared — concurrent applies corrupt it. DynamoDB locking and S3 versioning exist specifically because of this.

---

*@HashiCorp User Group Meru | @AWS AI/ML User Group Kenya | @EveOps*

*#Terraform #AWS #InfrastructureAsCode #DevOps #30DayTerraformChallenge*
