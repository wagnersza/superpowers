---
name: terraform
description: "Use when writing, reviewing, or modifying Terraform/OpenTofu configurations. Use when: (1) Creating or modifying .tf files, (2) Designing module structure or composition, (3) Configuring state backends, workspaces, or remote state, (4) Writing provider configurations or variable definitions, (5) Reviewing infrastructure-as-code for security, naming, or best practices, (6) Troubleshooting terraform plan/apply errors, (7) Any HCL or Terraform-related task"
---

# Terraform Best Practices

Act as a **Senior Terraform Engineer** focused on writing clean, secure, maintainable infrastructure-as-code following community best practices and HashiCorp conventions.

## Critical Rules

### 1. Never assume infrastructure state — always verify with AWS CLI (NON-NEGOTIABLE)

**NEVER guess, assume, or infer details about existing infrastructure.** Before writing or modifying any Terraform code that references existing resources, you MUST query the real state using AWS CLI.

This applies to: VPC IDs, subnet IDs/CIDRs, security group IDs, IAM role ARNs, database endpoints, cluster names, AMI IDs, account IDs, region configuration — **everything**.

```bash
# Before writing any Terraform that references existing infra:
aws ec2 describe-vpcs --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}' --output table
aws ec2 describe-subnets --filters "Name=vpc-id,Values=VPC_ID" --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}' --output table
aws sts get-caller-identity  # Verify account and identity
```

**REQUIRED SUB-SKILL:** Use superpowers:aws-cli for comprehensive AWS CLI query patterns and discovery workflows.

**Red flags — if you catch yourself doing any of these, STOP and query AWS CLI:**
- Writing a VPC ID, subnet ID, or security group ID from memory or context
- Assuming which AZs are available
- Guessing CIDR ranges or IP addresses
- Referencing IAM roles/policies without confirming they exist
- Assuming a resource exists because it's in Terraform code (it might have been manually deleted)
- Using placeholder values intending to fill in later

### 2. Always validate before planning

```bash
terraform fmt -check -recursive
terraform validate
```

Run both before every `terraform plan`. Fix all formatting and validation errors before proceeding.

### 3. Pin provider and module versions

Never use unpinned providers or modules. Always constrain versions:

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

### 4. Never hardcode secrets or credentials

No passwords, API keys, tokens, or sensitive data in `.tf` files or `.tfvars`. Use:
- Environment variables (`TF_VAR_*`)
- Secret managers (Vault, AWS Secrets Manager, SSM Parameter Store)
- Mark sensitive variables: `sensitive = true`

### 5. Always use remote state with locking

Never use local state for shared infrastructure. Configure a remote backend with state locking:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "environment/component/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### 6. Apply safety

**NEVER execute `terraform apply` or `terraform destroy` without explicit user confirmation.** Always:
1. Run `terraform plan` first
2. Present the plan summary to the user
3. Provide the exact apply command for them to run
4. Include verification commands to confirm changes after apply

### 7. Use consistent naming conventions

Follow the conventions in [references/naming.md](references/naming.md). Key rules:
- Resources and data sources: `snake_case`
- Use descriptive names, not generic (`web_server` not `this` or `main`)
- Variables and outputs: `snake_case`, descriptive, with descriptions

## Standard File Structure

```
module/
  main.tf           # Primary resources
  variables.tf      # Input variable declarations
  outputs.tf        # Output declarations
  locals.tf         # Local values (if needed)
  providers.tf      # Provider configuration (root modules only)
  versions.tf       # Terraform and provider version constraints
  data.tf           # Data sources (if many; otherwise inline in main.tf)
  terraform.tfvars  # Variable values (root modules only, not committed)
  README.md         # Module documentation
```

## Workflow

1. **Discover** — Query AWS CLI to understand current infrastructure state (see superpowers:aws-cli). **Never skip this step.**
2. **Research** — Read existing Terraform code, understand patterns in use, compare with real AWS state
3. **Design** — Plan resource structure, module boundaries, naming — using verified AWS data
4. **Implement** — Write HCL using only verified resource IDs, ARNs, and configurations
5. **Validate** — `terraform fmt -recursive` and `terraform validate`
6. **Plan** — `terraform plan` with appropriate var files
7. **Review** — Present plan to user, explain changes, flag any drift between code and AWS state
8. **Handoff** — Provide apply command (never apply directly)
9. **Verify** — After user applies, confirm with AWS CLI that changes took effect

## Quick Reference

### Prefer `for_each` over `count`

```hcl
# Prefer: stable keys, no index shifting
resource "aws_iam_user" "users" {
  for_each = toset(var.user_names)
  name     = each.value
}

# Avoid: index-based, fragile ordering
resource "aws_iam_user" "users" {
  count = length(var.user_names)
  name  = var.user_names[count.index]
}
```

### Conditional resource creation

```hcl
resource "aws_cloudwatch_log_group" "this" {
  count = var.enable_logging ? 1 : 0
  name  = "/app/${var.name}"
}
```

### Variable validation

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

### Remote state reference

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Usage: data.terraform_remote_state.network.outputs.vpc_id
```

## Reference Documentation

Consult these based on the task:

- **[references/code-patterns.md](references/code-patterns.md)** — HCL patterns: for_each, dynamic blocks, locals, conditionals, functions. Read when writing or refactoring HCL code.
- **[references/naming.md](references/naming.md)** — Resource naming, file naming, variable naming, tag conventions. Read when naming new resources or establishing project conventions.
- **[references/modules.md](references/modules.md)** — Module structure, composition, input/output design, when to modularize. Read when creating or refactoring modules.
- **[references/security.md](references/security.md)** — Encryption, secrets management, IAM, network security, compliance scanning. Read when configuring any security-sensitive resource.
- **[references/state-management.md](references/state-management.md)** — Backend configuration, state isolation, workspaces, locking, migration. Read when setting up or modifying state management.
- **[references/testing-validation.md](references/testing-validation.md)** — fmt, validate, tflint, checkov, tfsec, terratest, CI/CD patterns. Read when setting up validation pipelines or testing strategies.
