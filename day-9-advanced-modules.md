# Advanced Terraform Module Usage: Versioning, Gotchas, and Reuse Across Environments

*Day 9 of the 30-day IaC challenge — module gotchas that cause silent bugs, Git-based versioning, and deploying different module versions across dev and production*

---

Yesterday I built a reusable module. Today I learned the ways modules can silently break your infrastructure if you are not careful, and how to version them so your team can safely evolve infrastructure without surprises.

---

## The Three Module Gotchas

### Gotcha 1 — File paths inside modules

If your module references a file using a relative path, Terraform resolves it relative to where you run terraform, not relative to the module directory. This breaks silently.

Broken:
```terraform
user_data = base64encode(templatefile("user-data.sh", {
  server_port = var.server_port
}))
```

Correct:
```terraform
user_data = base64encode(templatefile("${path.module}/user-data.sh", {
  server_port = var.server_port
}))
```

`path.module` always resolves to the directory containing the module's own files, regardless of where Terraform is run from. This is the only safe way to reference files inside a module.

---

### Gotcha 2 — Inline blocks vs separate resources

Some AWS resources support both inline blocks and standalone resources for the same configuration. Security group ingress rules are the classic example. Mixing both causes conflicts.

Broken — mixing inline and separate:
```terraform
resource "aws_security_group" "alb" {
  name = "${var.cluster_name}-alb"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group_rule" "allow_http_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.alb.id
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

Correct — separate resources only:
```terraform
resource "aws_security_group" "alb" {
  name = "${var.cluster_name}-alb"
}

resource "aws_security_group_rule" "allow_http_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.alb.id
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

Separate resources are preferred in modules because callers can add rules from outside the module without modifying it.

---

### Gotcha 3 — Module output dependencies

If a root configuration uses `depends_on` referencing a module, Terraform treats the entire module as the dependency — not just the specific resource. This causes unnecessary recreation.

Broken:
```terraform
resource "aws_autoscaling_schedule" "scale_out" {
  depends_on             = [module.webserver_cluster]
  autoscaling_group_name = module.webserver_cluster.asg_name
}
```

Correct:
```terraform
resource "aws_autoscaling_schedule" "scale_out" {
  autoscaling_group_name = module.webserver_cluster.asg_name
}
```

Expose granular outputs from your module so callers never need `depends_on` at the module level.

---

## Module Versioning with Git Tags

The problem with local module paths is that any change to the module immediately affects every environment using it. The solution is versioning — push your module to a separate GitHub repository and tag each release:

```bash
git init
git add .
git commit -m "Initial commit of modules repo"
git remote add origin https://github.com/Mzokhulayo/terraform-aws-webserver-cluster.git
git push origin main
git tag -a "v0.0.1" -m "First release of webserver-cluster module"
git push origin main --tags
```

Make a change, commit and tag again:

```bash
git tag -a "v0.0.2" -m "Add instance_type output"
git push origin main --tags
```

Confirm both versions exist:

```bash
$ git tag -l
v0.0.1
v0.0.2
```

---

## The Three Module Source Types

Local path — for development only, no versioning:
```terraform
source = "../../../modules/services/webserver-cluster"
```

Git with version pin — for team use:
```terraform
source = "github.com/Mzokhulayo/terraform-aws-webserver-cluster//services/webserver-cluster?ref=v0.0.2"
```

Terraform Registry — for public or private registry modules:
```terraform
source  = "hashicorp/consul/aws"
version = "0.1.0"
```

The double slash `//` in the Git URL separates the repository from the subdirectory path within it. This is required when your module lives in a subdirectory rather than the repo root.

---

## Multi-Environment Version Pinning

Dev tests the new version. Production stays pinned to the last validated version.

**stage — testing v0.0.2:**
```terraform
module "webserver_cluster" {
  source = "github.com/Mzokhulayo/terraform-aws-webserver-cluster//services/webserver-cluster?ref=v0.0.2"

  cluster_name           = "webserver-stage"
  db_remote_state_bucket = "terraform-state-mzokhulayo"
  db_remote_state_key    = "stage/data-stores/mysql/terraform.tfstate"
  instance_type          = "t3.micro"
  min_size               = 2
  max_size               = 2
}
```

**prod — pinned to v0.0.1:**
```terraform
module "webserver_cluster" {
  source = "github.com/Mzokhulayo/terraform-aws-webserver-cluster//services/webserver-cluster?ref=v0.0.1"

  cluster_name           = "webserver-prod"
  db_remote_state_bucket = "terraform-state-mzokhulayo"
  db_remote_state_key    = "prod/data-stores/mysql/terraform.tfstate"
  instance_type          = "t3.micro"
  min_size               = 2
  max_size               = 4
}
```

Running `terraform init` in each environment downloads the correct version independently:

```
stage: Downloading git::https://github.com/Mzokhulayo/terraform-aws-webserver-cluster.git?ref=v0.0.2
prod:  Downloading git::https://github.com/Mzokhulayo/terraform-aws-webserver-cluster.git?ref=v0.0.1
```

Production never sees v0.0.2 until you explicitly update the ref and run `terraform init`.

---

## Why Version Pinning Matters

Without a version pin, the source URL points to the latest commit on the default branch. In a team environment, if engineer A pushes a change to the module and engineer B runs `terraform apply` five minutes later, B gets a different module than A ran. Their plans differ. Their state diverges. Production gets an untested change.

Version pins make `terraform apply` deterministic. The same ref always produces the same module. No surprises between runs.

---

*Next post: Terraform loops and conditionals — count, for_each, and dynamic blocks.*

*@HashiCorp User Group Meru | @AWS AI/ML User Group Kenya | @EveOps*

*#Terraform #AWS #InfrastructureAsCode #DevOps #30DayTerraformChallenge #CloudEngineering*
