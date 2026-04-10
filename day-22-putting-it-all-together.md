# Putting It All Together: Application and Infrastructure Workflows with Terraform

*Day 22 of the 30-day IaC challenge — the integrated pipeline, immutable plan promotion, Sentinel policies, and an honest reflection on 22 days of building.*

---

This is the day everything connects. Every concept from the past 22 days — version control, remote state, modules, testing, CI/CD, Sentinel policies — comes together into one coherent system. Today I built the integrated pipeline and wrote an honest reflection on the journey.

---

## The Integrated CI Pipeline

The final pipeline runs on every pull request in this sequence:

```yaml
name: Terraform Tests

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  validate:
    name: Validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.11.3"
      - name: Format check
        run: terraform fmt -check -recursive
        working-directory: modules/cluster/asg-rolling-deploy
      - name: Init
        run: terraform init -backend=false
        working-directory: modules/cluster/asg-rolling-deploy
      - name: Validate
        run: terraform validate
        working-directory: modules/cluster/asg-rolling-deploy

  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: validate
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.11.3"
      - name: Terraform Init
        run: terraform init
        working-directory: modules/cluster/asg-rolling-deploy
      - name: Run Unit Tests
        run: terraform test
        working-directory: modules/cluster/asg-rolling-deploy

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    needs: unit-tests
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_DEFAULT_REGION: us-east-1
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v4
        with:
          go-version: "1.21"
      - name: Run Integration Tests
        run: go test -v -timeout 30m -run TestHelloWorldAppExample ./...
        working-directory: Day-18-Automated-tests/tests
```

The format check caught real issues on the first run — `main.tf` and `outputs.tf` were not properly formatted. Fixed with `terraform fmt`. Pipeline went green in 41 seconds total.

Validate: 17s. Unit Tests: 16s. Integration Tests: skipped on PR, runs on merge to main only.

---

## Sentinel Policies

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

```hcl
import "tfplan/v2" as tfplan

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.change.after.tags["ManagedBy"] is "terraform"
  }
}
```

The instance type policy prevents engineers from accidentally deploying expensive instance types. The tagging policy ensures every resource can be identified as Terraform-managed — critical for cost attribution and drift detection.

---

## The Side-by-Side Comparison

| Component | Application Code | Infrastructure Code |
|---|---|---|
| Source of truth | Git repository | Git repository |
| Local run | npm start / python app.py | terraform plan |
| Artifact | Docker image / binary | Saved .tfplan file |
| Versioning | Semantic version tag | Semantic version tag |
| Automated tests | Unit + integration tests | terraform test + Terratest |
| Policy enforcement | Linting / SAST | Sentinel policies |
| Cost gate | N/A | Cost estimation policy |
| Promotion | Image promoted across envs | Plan promoted across envs |
| Deployment | CI/CD pipeline | terraform apply plan |
| Rollback | Redeploy previous image | terraform apply previous plan |

---

## Reflection on 22 Days

**What I built**

Remote state backends, S3 and DynamoDB locking, reusable modules with versioning, ASG with rolling deploy, ALB, RDS MySQL, Secrets Manager integration, CloudWatch alarms, zero-downtime deployments with create_before_destroy, blue/green deployments, loops and conditionals, manual testing with a structured checklist, automated testing with Terratest and terraform test, GitHub Actions CI/CD, HCP Terraform state migration, private module registry, Sentinel policies, and a complete seven-step deployment workflow end to end.

**What changed in how I think**

I used to think some amount of downtime was acceptable when deploying changes. The rolling deploy pattern changed that completely. The idea that at least one server stays healthy throughout the entire deployment — that Terraform can create the new resources before destroying the old ones — means downtime is a choice, not a requirement. That insight applies beyond Terraform to how I think about any system change.

**What was harder than expected**

Testing. Infrastructure testing requires deploying real resources, waiting for health checks, asserting against live endpoints, and destroying everything after. 15 minutes per integration test run. The gap between application testing and infrastructure testing is much larger than I expected. Writing the mock_provider block to satisfy preconditions without real AWS credentials took several iterations to get right.

**What I would do differently**

Nothing major — the challenge structure is correct. One chapter per day, one task at a time, code along with every example. The labs were sometimes difficult to find but working through them was worth it. I would not skip any day.

**What comes next**

Terraform Associate certification and AWS CloudOps Engineer exam. This challenge gave me the foundation — not just the commands, but the mental model for why infrastructure should be treated exactly like application code. Every concept I kept seeing in documentation but not fully understanding now makes sense.

---

*@HashiCorp User Group Meru | @AWS AI/ML User Group Kenya | @EveOps*

*#Terraform #AWS #InfrastructureAsCode #DevOps #TerraformCloud #30DayTerraformChallenge*
