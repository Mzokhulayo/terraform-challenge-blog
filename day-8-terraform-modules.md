# Building Reusable Infrastructure with Terraform Modules

*Day 8 of the 30-day IaC challenge — packaging a web server cluster into a reusable module and deploying it across environments with zero code duplication*

---

By Day 5 I had a working web server cluster — ASG, ALB, security groups, launch template, remote state. It worked. But it was a flat collection of files tied to one environment. If I wanted a production version, I would copy the entire directory and change a few values. That is the problem modules solve.

A Terraform module is just a directory of Terraform files. What makes it a module is how you use it — you call it from another configuration and pass inputs. The module handles the implementation. The caller handles the environment-specific values.

---

## The Module Directory Structure

```
modules/
└── services/
    └── webserver-cluster/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        ├── user-data.sh
        └── README.md

stage/
└── services/
    └── webserver-cluster/
        ├── main.tf
        └── outputs.tf
```

The modules directory is the library. The stage directory is the consumer. You never run `terraform apply` inside the module — only inside the calling configuration.

---

## The Module Code

### variables.tf

```terraform
variable "cluster_name" {
  description = "The name to use for all the cluster resources"
  type        = string
}

variable "db_remote_state_bucket" {
  description = "The name of the remote bucket for the database's remote state"
  type        = string
}

variable "db_remote_state_key" {
  description = "The path for the database's remote state in S3"
  type        = string
}

variable "instance_type" {
  description = "The type of instance to run"
  type        = string
}

variable "min_size" {
  description = "The minimum number of EC2 instances in the ASG"
  type        = number
}

variable "max_size" {
  description = "The maximum number of EC2 instances in the ASG"
  type        = number
}

variable "server_port" {
  description = "The port the server will use for HTTP requests"
  type        = number
  default     = 8080
}

variable "ami_id" {
  description = "AMI ID for the EC2 instance"
  type        = string
  default     = "ami-0c02fb55956c7d316"
}
```

`cluster_name` has no default because every environment must name its resources explicitly. `min_size` and `max_size` have no defaults because forcing the caller to be explicit is safer than a default that gets forgotten. `server_port` defaults to 8080 because that is the established convention across this project.

### outputs.tf

```terraform
output "alb_dns_name" {
  value       = aws_alb.example.dns_name
  description = "The domain name of the load balancer"
}

output "asg_name" {
  value       = aws_autoscaling_group.example.name
  description = "The name of the Auto Scaling Group"
}

output "alb_security_group_id" {
  value       = aws_security_group.alb.id
  description = "The ID of the security group attached to the load balancer"
}
```

`alb_dns_name` is the primary output. `asg_name` is useful for attaching scheduled scaling actions from outside the module. `alb_security_group_id` lets a caller add ingress rules without modifying the module itself.

---

## The Calling Configuration

```terraform
terraform {
  backend "s3" {
    bucket         = "terraform-state-mzokhulayo"
    key            = "stage/services/webserver-cluster/terraform.tfstate"
    region         = "us-east-2"
    dynamodb_table = "terraform-up-and-running-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = "us-east-1"
}

module "webserver_cluster" {
  source = "../../../modules/services/webserver-cluster"

  cluster_name           = "webserver-stage"
  db_remote_state_bucket = "terraform-state-mzokhulayo"
  db_remote_state_key    = "stage/data-stores/mysql/terraform.tfstate"
  instance_type          = "t3.micro"
  min_size               = 2
  max_size               = 2
}
```

The `source` path is relative — three levels up to the repo root, then into the module.

---

## Deployment Confirmation

```
$ terraform output
alb_dns_name = "terraform-asg-example-1971246899.us-east-1.elb.amazonaws.com"

$ curl http://terraform-asg-example-1971246899.us-east-1.elb.amazonaws.com
<h1>Hello World By Mzokhulayo Mdubeki</h1>
<h1>Project: My 30-Days Terraform Learning</h1>
<p>DB address: terraform-up-and-running20260322143042906300000001.c52m8ecgkpwl.us-east-2.rds.amazonaws.com</p>
<p>DB port: 3306</p>
```

---

## What Makes a Module Easy to Use

A module is easy to use when the caller only needs to know what the infrastructure does, not how it works. The caller passes `cluster_name`, `instance_type`, and sizes — it never touches security group rules, launch templates, or health check configuration. All of that is internal.

A module is painful when it exposes too many variables or too few. The right balance is exposing what changes per environment and hiding what stays the same. Missing outputs are also painful — if the caller needs the ALB security group ID but the module does not output it, the caller is stuck.

---

## Best Practices

**Never put a backend block inside a module.** Terraform ignores it with a warning but it creates confusion. Backend configuration belongs only in root modules.

**Use locals for repeated values.** The `http_port`, `any_port`, `tcp_protocol`, and `all_ips` values appear in multiple resources — defining them as locals once makes the module easier to update.

**Write a README.** A usage example, an inputs table, and an outputs table is enough. Anyone picking up your module in six months will thank you.

---

*Next post: Terraform loops and conditionals — how to avoid repeating resource blocks.*

*@HashiCorp User Group Meru | @AWS AI/ML User Group Kenya | @EveOps*

*#Terraform #AWS #InfrastructureAsCode #DevOps #30DayTerraformChallenge #CloudEngineering*
