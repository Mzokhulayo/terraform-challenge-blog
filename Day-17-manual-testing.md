# The Importance of Manual Testing in Terraform

*Day 17 of the 30-day IaC challenge — building a structured test checklist, executing it against a live deployment, and establishing the cleanup discipline that keeps costs from spiraling*

---

Testing is what separates engineers who hope their infrastructure works from engineers who know it does. Before you can write automated tests, you need to know exactly what you are testing, why, and how to verify it. Today I built a structured manual testing process for the webserver cluster, documented every result, and established the cleanup habit that has to come after every test run.

---

## Why Manual Testing Still Matters

Automated tests are faster and repeatable, but they test what you tell them to test. Manual testing catches the things you did not think to automate — a misconfigured security group rule that still allows traffic, a health check that passes but returns unexpected content, an ASG that creates instances in the wrong region because a provider alias was left in place.

The author's point in Chapter 9 is direct: when testing Terraform code, you cannot use localhost. The only practical way to manually test is to deploy to a real environment, verify it works, and destroy it. That is exactly what this day covers.

---

## The Test Checklist

### Provisioning Verification

- Does `terraform init` complete without errors?
- Does `terraform validate` pass cleanly?
- Does `terraform plan` show the expected number and type of resources?
- Does `terraform apply` complete without errors?

### Resource Correctness

- Are all expected resources visible in AWS?
- Do resource names, regions, and security group rules match configuration?
- Are both instances registered in the target group?

### Functional Verification

- Does the ALB DNS name resolve?
- Does `curl http://<alb-dns>` return the expected response?
- Do all instances pass health checks?

### State Consistency

- Does `terraform plan` return "No changes" immediately after apply?
- Does the state file accurately reflect what exists in AWS?

### Regression Check

- Make a small change. Does `terraform plan` show only that change?
- Apply it. Does `terraform plan` return clean afterward?

---

## Test Execution Results

**Test: terraform init completes successfully**
Command: `terraform init`
Expected: Successful initialization with modules downloaded
Actual: Successfully initialized — modules resolved from local paths
Result: PASS

**Test: terraform validate passes**
Command: `terraform validate`
Expected: `Success! The configuration is valid.`
Actual: `Success! The configuration is valid.`
Result: PASS — after fixing a typo (`db_adress` → `db_address`) and removing duplicate output definitions

**Test: terraform plan shows expected resources**
Command: `terraform plan`
Expected: 13 resources to create
Actual: 13 resources to create — ALB, listener, listener rule, target group, 2 security groups, 3 security group rules, launch template, ASG, 2 CloudWatch alarms
Result: PASS

**Test: terraform apply completes without errors**
Command: `terraform apply -auto-approve`
Expected: Apply complete with ALB DNS name output
Actual: `Apply complete! Resources: 13 added, 0 changed, 0 destroyed.`
ALB DNS: `hello-world-stage-757244467.us-east-1.elb.amazonaws.com`
Result: PASS

**Test: ALB returns expected response**
Command: `curl -s http://hello-world-stage-757244467.us-east-1.elb.amazonaws.com`
Expected: Hello World v-1 with DB address
Actual:
```
<h1>Hello World v-1</h1>
<h1>Project: My 30-Days Terraform Learning</h1>
<p>DB address: terraform-up-and-running20260407021643970200000001.ca1cu4weqvjl.us-east-1.rds.amazonaws.com</p>
<p>DB port: 3306</p>
```
Result: PASS

**Test: Both instances healthy in target group**
Command: `aws elbv2 describe-target-health ...`
Expected: All instances in healthy state
Actual: i-06c1d1d6941e2bbb5 healthy, i-096ae5a6b887f087d healthy
Result: PASS

**Test: terraform plan returns No changes after apply**
Command: `terraform plan`
Expected: No changes
Actual: `No changes. Your infrastructure matches the configuration.`
Result: PASS

**Test: Regression — adding a tag shows only that change**
Command: Added `custom_tags = { TestTag = "manual-test-day17" }`, ran `terraform plan`
Expected: 1 resource to change — ASG tag update only
Actual: `Plan: 0 to add, 1 to change, 0 to destroy` — only the ASG tag update
Result: PASS

---

## Failures Found and Fixed

**Failure 1: Duplicate output definitions**

`modules/services/hello-world-app/outputs.tf` had two sets of outputs — one referencing direct resource attributes (`aws_alb.example`, `aws_autoscaling_group.example`) that did not exist in the module, and one correctly referencing module outputs. Terraform errored with `Duplicate output definition`.

Fix: removed the old direct resource references and kept only the module-based outputs.

**Failure 2: Typo in templatefile call**

`db_adress` instead of `db_address` in the templatefile vars map caused validate to fail with `vars map does not contain key "db_address"`.

Fix: corrected the spelling.

**Failure 3: Provider alias preventing region override**

The provider block had `alias = "region_1"` which meant it was not the default provider. All resources deployed to `af-south-1` instead of `us-east-1`.

Fix: removed the alias so the provider became the default.

**Failure 4: Wrong backend key on DB state**

`live/stage/data-stores/mysql/main.tf` had `key = "stage/services/webserver-cluster/terraform.tfstate"` — it was saving DB state to the webserver-cluster path. When the webserver-cluster tried to read `stage/data-stores/mysql/terraform.tfstate`, it found an empty state with no outputs.

Fix: corrected the key to `stage/data-stores/mysql/terraform.tfstate` and ran `terraform init -reconfigure` followed by `terraform apply`.

**Failure 5: Invalid base64 user data**

The launch template received raw user data instead of base64 encoded. AWS rejected it with `InvalidUserData.Malformed`.

Fix: wrapped the `templatefile()` call with `base64encode()` in `modules/services/hello-world-app/main.tf`.

---

## Cleanup Verification

After `terraform destroy -auto-approve` completed with `Destroy complete! Resources: 13 destroyed`:

```bash
# No running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --region us-east-1 \
  --query 'Reservations[*].Instances[*].[InstanceId]' \
  --output table
# Returns empty

# No load balancers
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query 'LoadBalancers[*].LoadBalancerName' \
  --output text
# Returns empty
```

Clean destroy confirmed.

---

## Chapter 9 Takeaways

Cleaning up after tests is harder than it sounds because Terraform destroy can fail partway through — leaving orphaned security groups, target groups, or instances that continue accruing costs. The ASG in this test took nearly 6 minutes to destroy because it had to drain and terminate instances first. If that had failed, the instances would have kept running silently.

The risk of not cleaning up is not just cost — it is state pollution. On the next apply, Terraform may find name conflicts with leftover resources, causing apply to fail in ways that are hard to debug.

The terraform import lab showed how to bring existing resources under Terraform management without recreating them. `terraform import` solves the problem of resources that were created manually or by another tool — it pulls their current state into the state file so Terraform can manage them going forward. What it does not do is generate the configuration — you still have to write the resource block manually to match what was imported.

---

*Next post: Automated testing with Terratest — unit tests, integration tests, and end-to-end tests for Terraform modules.*

*@HashiCorp User Group Meru | @AWS AI/ML User Group Kenya | @EveOps*

*#Terraform #AWS #InfrastructureAsCode #Testing #DevOps #30DayTerraformChallenge*
