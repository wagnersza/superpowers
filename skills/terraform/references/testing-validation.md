# Terraform Testing and Validation

## Table of Contents

- [TDD with Terraform Test](#tdd-with-terraform-test)
- [Test Scope Rules](#test-scope-rules)
- [Test Patterns by Resource Type](#test-patterns-by-resource-type)
- [Built-in Validation](#built-in-validation)
- [Linting with tflint](#linting-with-tflint)
- [Security Scanning](#security-scanning)
- [Plan Review Practices](#plan-review-practices)
- [Local Validation Workflow](#local-validation-workflow)

## TDD with Terraform Test

**`terraform test` is the primary testing tool.** Follow the RED-GREEN-REFACTOR cycle from superpowers:test-driven-development:

### The TDD Cycle for Terraform

```
1. RED    — Write .tftest.hcl with assertions for the new resource
           Run `terraform test` → FAIL (resource doesn't exist yet)

2. GREEN  — Write minimal .tf code to make the test pass
           Run `terraform test` → PASS

3. REFACTOR — Clean up, improve naming, extract modules
             Run `terraform test` → still PASS
```

### Writing tests FIRST

```hcl
# tests/s3_data_bucket.tftest.hcl — Write THIS before any resource code

variables {
  project     = "myapp"
  environment = "test"
}

run "verify_bucket_naming_and_security" {
  command = plan  # Plan mode — no real resources, no cost

  # Naming convention
  assert {
    condition     = aws_s3_bucket.data.bucket == "myapp-test-data"
    error_message = "Bucket name must follow {project}-{env}-{purpose} convention"
  }

  # Security: public access blocked
  assert {
    condition     = aws_s3_bucket_public_access_block.data.block_public_acls == true
    error_message = "Public ACLs must be blocked"
  }

  assert {
    condition     = aws_s3_bucket_public_access_block.data.block_public_policy == true
    error_message = "Public policy must be blocked"
  }

  # Security: encryption enabled
  assert {
    condition     = aws_s3_bucket_server_side_encryption_configuration.data.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm == "aws:kms"
    error_message = "Must use KMS encryption"
  }

  # Versioning enabled
  assert {
    condition     = aws_s3_bucket_versioning.data.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning must be enabled"
  }
}
```

### Running tests

```bash
# Run all tests
terraform test

# Run specific test file
terraform test -filter=tests/s3_data_bucket.tftest.hcl

# Verbose output (shows each assertion)
terraform test -verbose
```

### Test file structure

```
module/
  main.tf
  variables.tf
  outputs.tf
  tests/
    s3_data_bucket.tftest.hcl      # One file per logical resource group
    iam_service_role.tftest.hcl
    ecs_service.tftest.hcl
```

### Plan mode vs Apply mode

```hcl
# PLAN MODE (preferred) — fast, free, no real resources
run "verify_config" {
  command = plan
  assert { ... }
}

# APPLY MODE — creates real resources, use only when you must verify runtime behavior
run "verify_runtime" {
  command = apply
  assert {
    condition     = output.endpoint != ""
    error_message = "Endpoint must be available after creation"
  }
}
```

**Default to `command = plan`** — it catches most configuration issues without cost or risk. Use `command = apply` only when you need to verify outputs, computed values, or cross-resource behavior that plan cannot evaluate.

## Test Scope Rules

### MUST test (new resources)

Write tests for ALL new resources before writing the resource code:

- Naming convention compliance
- Security configuration (encryption, public access, IAM)
- Required tags
- Variable validation (correct types, constraints)
- Conditional logic (feature flags, count/for_each)
- Output values

### Do NOT test (unless asked)

- **Existing/old infrastructure** — don't retroactively add tests
- **Resources you didn't create or modify** — not your scope
- **Third-party module internals** — test your usage, not the module's implementation

### Test when modifying existing resources

When changing an existing resource, write tests **only for the specific changes**:

```hcl
# Changing encryption from AES256 to KMS — test ONLY the encryption change
run "verify_encryption_upgrade" {
  command = plan

  assert {
    condition     = aws_s3_bucket_server_side_encryption_configuration.data.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm == "aws:kms"
    error_message = "Encryption must be upgraded to KMS"
  }
}
```

## Test Patterns by Resource Type

### Networking (VPC, subnets, security groups)

```hcl
run "verify_vpc_config" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR must be 10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames must be enabled"
  }
}

run "verify_private_subnets" {
  command = plan

  assert {
    condition     = length(aws_subnet.private) == 3
    error_message = "Must have 3 private subnets (one per AZ)"
  }
}

run "verify_no_open_ssh" {
  command = plan

  assert {
    condition     = !contains([for r in aws_security_group_rule.ingress : r.cidr_blocks], ["0.0.0.0/0"]) || !contains([for r in aws_security_group_rule.ingress : r.from_port], 22)
    error_message = "SSH must not be open to 0.0.0.0/0"
  }
}
```

### IAM roles and policies

```hcl
run "verify_least_privilege" {
  command = plan

  # Verify no wildcard actions
  assert {
    condition     = !strcontains(data.aws_iam_policy_document.task.json, "\"Action\":\"*\"")
    error_message = "IAM policy must not use wildcard actions"
  }

  # Verify no wildcard resources
  assert {
    condition     = !strcontains(data.aws_iam_policy_document.task.json, "\"Resource\":\"*\"")
    error_message = "IAM policy must not use wildcard resources"
  }
}
```

### Database (RDS)

```hcl
run "verify_database_security" {
  command = plan

  assert {
    condition     = aws_db_instance.main.storage_encrypted == true
    error_message = "Database storage must be encrypted"
  }

  assert {
    condition     = aws_db_instance.main.publicly_accessible == false
    error_message = "Database must not be publicly accessible"
  }

  assert {
    condition     = aws_db_instance.main.deletion_protection == true
    error_message = "Deletion protection must be enabled"
  }

  assert {
    condition     = aws_db_instance.main.backup_retention_period >= 7
    error_message = "Backup retention must be at least 7 days"
  }
}
```

### Tags

```hcl
run "verify_required_tags" {
  command = plan

  assert {
    condition     = aws_s3_bucket.data.tags["Environment"] == var.environment
    error_message = "Environment tag must be set"
  }

  assert {
    condition     = aws_s3_bucket.data.tags["ManagedBy"] == "terraform"
    error_message = "ManagedBy tag must be 'terraform'"
  }

  assert {
    condition     = aws_s3_bucket.data.tags["Project"] != null
    error_message = "Project tag must be set"
  }
}
```

### Outputs

```hcl
run "verify_outputs" {
  command = plan

  assert {
    condition     = output.bucket_arn != null
    error_message = "bucket_arn output must be defined"
  }

  assert {
    condition     = output.bucket_name == "myapp-test-data"
    error_message = "bucket_name output must match expected value"
  }
}
```

## Built-in Validation

### Always run before plan

```bash
# Format check (fails if files need formatting)
terraform fmt -check -recursive

# Auto-format all files
terraform fmt -recursive

# Validate configuration syntax and internal consistency
terraform validate
```

### Variable validation rules

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "cidr_block" {
  type        = string
  description = "CIDR block for the VPC"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}
```

### Pre-conditions and post-conditions (Terraform 1.2+)

```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type
  ami           = var.ami_id

  lifecycle {
    precondition {
      condition     = data.aws_ami.selected.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
}
```

## Linting with tflint

### Installation and configuration

```bash
# Install
brew install tflint           # macOS
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash  # Linux

# Initialize plugins
tflint --init

# Run
tflint --recursive
```

### Configuration file (`.tflint.hcl`)

```hcl
config {
  call_module_type = "local"
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "aws" {
  enabled = true
  version = "0.31.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

# Custom rules
rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}
```

### Common tflint rules

| Rule | What it catches |
|------|----------------|
| `terraform_naming_convention` | Non-snake_case identifiers |
| `terraform_documented_variables` | Variables without descriptions |
| `terraform_documented_outputs` | Outputs without descriptions |
| `terraform_unused_declarations` | Unused variables, data sources |
| `aws_instance_invalid_type` | Non-existent EC2 instance types |
| `aws_resource_missing_tags` | Resources without required tags |

## Security Scanning

### checkov

```bash
# Install
pip install checkov

# Scan directory
checkov -d .

# Scan with specific framework
checkov -d . --framework terraform

# Skip specific checks
checkov -d . --skip-check CKV_AWS_18,CKV_AWS_21

# Output formats
checkov -d . -o json
checkov -d . -o junitxml > results.xml
```

### tfsec / trivy

```bash
# trivy (successor to tfsec)
brew install trivy
trivy config .

# With specific severity
trivy config --severity HIGH,CRITICAL .

# Ignore specific rules
trivy config --skip-dirs modules/legacy .
```

### Inline suppression

```hcl
resource "aws_s3_bucket" "public_assets" {
  bucket = "public-assets"

  #checkov:skip=CKV_AWS_18:Public bucket for static assets by design
  #trivy:ignore:AVD-AWS-0086
}
```

Only suppress checks with a clear justification comment.

### When to use which testing approach

| Approach | Speed | Cost | Best For |
|----------|-------|------|----------|
| `terraform test` (plan) | Seconds | Free | **Primary — TDD for all new resources** |
| `terraform validate` | Instant | Free | Syntax errors, type mismatches |
| `tflint` | Fast | Free | Best practice violations, invalid values |
| `checkov`/`trivy` | Fast | Free | Security policy compliance |
| `terraform test` (apply) | Minutes | Real cost | Runtime behavior verification (use sparingly) |

## Plan Review Practices

### Always review plan output

```bash
# Generate plan file
terraform plan -out=tfplan

# Review in detail
terraform show tfplan

# JSON output for programmatic review
terraform show -json tfplan | jq '.resource_changes[] | {action: .change.actions, address: .address}'
```

### What to look for in plan output

| Signal | Meaning | Action |
|--------|---------|--------|
| `+ create` | New resource | Verify it's intended |
| `~ update in-place` | Modify existing | Check what's changing |
| `-/+ must replace` | Destroy and recreate | Check for downtime, data loss |
| `- destroy` | Delete resource | Verify it's intended, check dependencies |
| `<= read` | Data source refresh | Usually safe |

### Red flags in plan output

- **Unexpected destroys** — especially databases, storage, stateful resources
- **Force replacement** on resources that should be updated in-place
- **Large number of changes** when you only modified a few resources
- **Changes to resources you didn't touch** — may indicate state drift

## Local Validation Workflow

Run this sequence locally before handing off to the user for apply. This mirrors the superpowers workflow: discover → test → implement → validate → plan → handoff.

```bash
# 1. Format
terraform fmt -recursive

# 2. Validate syntax
terraform validate

# 3. Run tests (TDD — should already be passing from your RED-GREEN-REFACTOR cycle)
terraform test

# 4. Lint (if tflint is available)
tflint --recursive

# 5. Security scan (if checkov/trivy is available)
checkov -d . --framework terraform

# 6. Plan — review output carefully
terraform plan -var-file=prod.tfvars

# 7. Handoff — present plan and apply command to user (NEVER apply yourself)
```

**This is a local developer workflow, not a CI/CD pipeline.** You run these steps interactively during development. The human operator makes the final decision to apply.
