# Terraform Testing and Validation

## Table of Contents

- [Built-in Validation](#built-in-validation)
- [Linting with tflint](#linting-with-tflint)
- [Security Scanning](#security-scanning)
- [Terraform Test Framework](#terraform-test-framework)
- [Integration Testing with Terratest](#integration-testing-with-terratest)
- [Plan Review Practices](#plan-review-practices)
- [CI/CD Pipeline Patterns](#cicd-pipeline-patterns)

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

## Terraform Test Framework

Native testing (Terraform 1.6+) using `.tftest.hcl` files:

### Unit test (no real resources)

```hcl
# tests/naming.tftest.hcl
variables {
  project     = "myapp"
  environment = "dev"
}

run "verify_naming" {
  command = plan

  assert {
    condition     = aws_s3_bucket.this.bucket == "myapp-dev-data"
    error_message = "Bucket name should follow naming convention"
  }

  assert {
    condition     = aws_s3_bucket.this.tags["Environment"] == "dev"
    error_message = "Environment tag should match variable"
  }
}
```

### Integration test (creates real resources)

```hcl
# tests/integration.tftest.hcl
variables {
  project     = "test"
  environment = "test"
}

run "create_infrastructure" {
  command = apply

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC should be created"
  }
}

run "verify_network" {
  command = plan

  module {
    source = "./tests/verify-network"
  }
}
```

### Running tests

```bash
# Run all tests
terraform test

# Run specific test file
terraform test -filter=tests/naming.tftest.hcl

# Verbose output
terraform test -verbose
```

## Integration Testing with Terratest

For complex infrastructure requiring real resource verification:

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVpc(t *testing.T) {
    t.Parallel()

    opts := &terraform.Options{
        TerraformDir: "../modules/networking",
        Vars: map[string]interface{}{
            "environment": "test",
            "vpc_cidr":    "10.99.0.0/16",
        },
    }

    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    vpcId := terraform.Output(t, opts, "vpc_id")
    assert.NotEmpty(t, vpcId)

    privateSubnets := terraform.OutputList(t, opts, "private_subnet_ids")
    assert.Equal(t, 3, len(privateSubnets))
}
```

### When to use which testing approach

| Approach | Speed | Cost | Best For |
|----------|-------|------|----------|
| `terraform validate` | Instant | Free | Syntax errors, type mismatches |
| `tflint` | Fast | Free | Best practice violations, invalid values |
| `checkov`/`trivy` | Fast | Free | Security policy compliance |
| `terraform test` (plan) | Seconds | Free | Logic verification without provisioning |
| `terraform test` (apply) | Minutes | Real cost | End-to-end with real resources |
| Terratest | Minutes | Real cost | Complex multi-step infrastructure tests |

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

## CI/CD Pipeline Patterns

### GitHub Actions example

```yaml
name: Terraform
on:
  pull_request:
    paths: ['terraform/**']
  push:
    branches: [main]
    paths: ['terraform/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Format Check
        run: terraform fmt -check -recursive
        working-directory: terraform

      - name: Init
        run: terraform init -backend=false
        working-directory: terraform

      - name: Validate
        run: terraform validate
        working-directory: terraform

      - name: tflint
        uses: terraform-linters/setup-tflint@v4
      - run: |
          tflint --init
          tflint --recursive
        working-directory: terraform

      - name: Security Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: terraform
          severity: HIGH,CRITICAL

  plan:
    needs: validate
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Plan
        run: |
          terraform init
          terraform plan -no-color -out=tfplan
        working-directory: terraform

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            // Post plan output as PR comment
```

### Pipeline stages

```
PR opened/updated:
  1. terraform fmt -check
  2. terraform validate
  3. tflint
  4. checkov / trivy
  5. terraform plan
  6. Post plan as PR comment

PR merged to main:
  7. terraform plan (verify)
  8. Manual approval gate
  9. terraform apply
  10. Post-apply verification
```

### Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_docs
      - id: terraform_checkov
```

```bash
# Install and run
pip install pre-commit
pre-commit install
pre-commit run --all-files
```
