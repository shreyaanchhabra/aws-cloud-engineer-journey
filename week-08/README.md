# Week 8: Terraform VPC

Reproduces the Week 5 VPC using Terraform and the `terraform-aws-modules/vpc/aws`
module (v5.8.1).

- Name: `training-vpc-tf`
- CIDR: `10.1.0.0/16`
- 2 public subnets and 2 private subnets across `us-east-1a` and `us-east-1b`
- 1 NAT gateway (single, shared)

## What each command does

### `terraform plan`
Reads my `.tf` code, compares it to the state file and to what actually exists
in AWS, and prints what Terraform *would* create, change, or destroy. It changes
nothing. It's the safety check: I read every line before applying, looking for
unexpected `destroy` or `replace` actions.

### `terraform apply`
Shows the plan again, asks for `yes`, then calls the AWS APIs to create or
modify the resources. When it finishes, it records the real resource IDs in the
state file. This is the only command in the workflow that creates things (and
starts NAT gateway charges).

### `terraform destroy`
Deletes every resource Terraform manages in this configuration, in dependency
order, and removes them from state. I run it at the end of each session to stop
NAT gateway and Elastic IP charges.

## Why state matters

The state file (`terraform.tfstate`) is Terraform's record of which real AWS
resources correspond to which blocks in my code. Everything depends on it:

- **Mapping:** without state, Terraform can't tell that the VPC in my code is
  the VPC `vpc-0abc...` already in AWS, so it would try to create duplicates.
- **Planning:** `plan` is a three-way comparison of code, state, and reality.
  Diffs between them are how it knows what to do.
- **Drift detection:** if someone changes a resource outside Terraform, state
  and reality disagree and `plan` surfaces it.
- **Dependency tracking:** it lets `destroy` tear things down in the right order.

State can contain sensitive values, so `terraform.tfstate` and `.terraform/`
are in `.gitignore` and never committed. In a team setting, state goes in a
remote backend (for example S3 with locking) instead of on one laptop.
