# A Workflow for Deploying Application Code with Terraform

*Day 20 of the 30-day IaC challenge — seven steps from local change to production, HCP Terraform state migration, and the private module registry.*

---

Good software teams have a deployment workflow. Every change goes through version control, local testing, code review, automated tests, and a controlled deployment. Terraform infrastructure changes should go through exactly the same process. Today I walked all seven steps end to end using the webserver cluster built over the past 19 days.

---

## Step 1 - Version Control

The Terraform configuration lives in Git, the same as application code. The state file does not. That distinction matters — the state file contains sensitive values and changes on every apply. It belongs in a remote backend, never in the repository.

Main branch is protected. No direct pushes. All changes come through pull requests.

---

## Step 2 - Run Locally

Before making any change, run plan and save the output:

```bash
terraform plan -out=day20.tfplan
```

Saving the plan guarantees that what you reviewed is exactly what gets applied. Never apply without reviewing the plan first.

The change was updating `server_text` from `"Hello World v-1"` to `"Hello World V3"`. The plan showed a launch template update with new user data — exactly what was expected.

```
Plan: 13 to add, 0 to change, 0 to destroy.
Saved the plan to: day20.tfplan
```

---

## Step 3 - Make the Code Change

```bash
git checkout -b update-app-version-day20
git add .
git commit -m "Update app response to Hello World V3 for Day 20"
git push origin update-app-version-day20
```

Feature branch per change. Every infrastructure change should be isolated and reviewable independently.

---

## Step 4 - Submit for Review

Open a pull request and paste the terraform plan output as a comment. This is the infrastructure equivalent of a code diff. The reviewer sees exactly what will change in AWS without running Terraform themselves.

This is the step most teams skip. Without the plan output in the PR, reviewers are approving changes they cannot fully evaluate.

PR: [Update app response to Hello World V3 for Day 20 #1](https://github.com/Mzokhulayo/30-Days-Terraform-Challenge/pull/1)

---

## Step 5 - Automated Tests

GitHub Actions triggered automatically on the PR. Unit tests passed in 15 seconds. Integration tests were skipped on PR — they only run on push to main. The PR was ready to merge.

```
Terraform Tests / Unit Tests (pull_request)   Successful in 15s
Terraform Tests / Integration Tests           Skipped
```

---

## Step 6 - Merge and Release

Merged the PR then tagged the merge commit:

```bash
git tag -a "v1.3.0" -m "Update app response to Hello World V3"
git push origin v1.3.0
```

Tags give you a precise point to roll back to if a deployment causes problems.

---

## Step 7 - Deploy

Applied the saved plan:

```bash
terraform apply "day20.tfplan"
```

Verified the change was live:

```bash
curl http://$(terraform output -raw alb_dns_name)
```

```html
<h1>Hello World V3</h1>
<h1>Project: My 30-Days Terraform Learning</h1>
<p>DB address: terraform-up-and-running20260409205516324500000001.ca1cu4weqvjl.us-east-1.rds.amazonaws.com</p>
<p>DB port: 3306</p>
```

---

## HCP Terraform - State Migration

After the seven steps, I migrated state from S3 to HCP Terraform. The migration happens during init:

```bash
terraform init
# Migrating from backend "s3" to HCP Terraform
# Should Terraform migrate your existing state? yes
# HCP Terraform has been successfully initialized!
```

The terraform block changes from a backend block to a cloud block:

```terraform
terraform {
  cloud {
    organization = "Mzokhulayo-Mdubeki"

    workspaces {
      name = "webserver-cluster-stage"
    }
  }
}
```

AWS credentials are now stored as sensitive environment variables in the workspace — not on any developer machine, not in any CI config file. Every run is logged with who triggered it, the full plan output, and what changed.

---

## Private Registry

The webserver-cluster module is published to the HCP Terraform private registry at version 0.0.2. Any team member in the organisation references it as:

```terraform
module "webserver_cluster" {
  source  = "app.terraform.io/Mzokhulayo-Mdubeki/webserver-cluster/aws"
  version = "0.0.2"
}
```

This is more reliable than a direct GitHub URL because the registry enforces versioning, hosts documentation generated from the README, and prevents consumers from accidentally pulling an unpinned latest commit.

---

## Workflow Comparison

| Step | Application Code | Infrastructure Code | Key Difference |
|---|---|---|---|
| Version control | Git for source | Git for .tf files | State file is NOT in Git |
| Run locally | Start the app | terraform plan | Plan shows future changes, not a running app |
| Make changes | Edit source files | Edit .tf files | Changes affect real cloud resources |
| Review | Code diff in PR | Plan output in PR | Reviewer must understand cloud implications |
| Automated tests | Unit tests, linting | terraform test, Terratest | Infra tests deploy real resources and cost money |
| Merge and release | Merge and tag | Merge and tag | Module consumers must pin to versions |
| Deploy | CI/CD pipeline | terraform apply | Apply must run from a trusted environment |

The biggest difference is step 5. Application unit tests run in milliseconds at zero cost. Infrastructure integration tests deploy real AWS resources, take 15 minutes, and cost money. This forces a different testing strategy — fast unit tests on every PR, expensive integration tests only on merge to main.

---

*@HashiCorp User Group Meru | @AWS AI/ML User Group Kenya | @EveOps*

*#Terraform #AWS #InfrastructureAsCode #DevOps #TerraformCloud #30DayTerraformChallenge*
