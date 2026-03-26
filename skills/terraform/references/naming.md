# Terraform Naming Conventions

## Table of Contents

- [Resource Naming](#resource-naming)
- [File Naming](#file-naming)
- [Variable Naming](#variable-naming)
- [Output Naming](#output-naming)
- [Module Naming](#module-naming)
- [Tag Standards](#tag-standards)

## Resource Naming

### HCL identifiers (in code)

Use `snake_case` for all Terraform identifiers:

```hcl
# Resources and data sources
resource "aws_security_group" "web_server" { }
data "aws_ami" "amazon_linux_2023" { }

# Variables and locals
variable "instance_type" { }
locals { name_prefix = "..." }

# Outputs
output "instance_public_ip" { }
```

### Resource naming rules

| Rule | Example | Anti-pattern |
|------|---------|-------------|
| Descriptive names | `aws_instance.web_server` | `aws_instance.this` (unless single resource in module) |
| Use `this` only for primary resource in a module | `aws_s3_bucket.this` (in s3 module) | `aws_s3_bucket.this` (in root module with multiple buckets) |
| No resource type in name | `aws_instance.web` | `aws_instance.web_instance` |
| No provider in name | `aws_s3_bucket.logs` | `aws_s3_bucket.aws_logs` |
| Plural for collections | `aws_iam_user.developers` | `aws_iam_user.developer` (when using for_each) |

### Cloud resource names (deployed names)

Use a consistent pattern: `{project}-{environment}-{component}`

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs"
  # Result: myapp-prod-logs
}

resource "aws_ecs_cluster" "main" {
  name = "${local.name_prefix}-cluster"
  # Result: myapp-prod-cluster
}
```

## File Naming

### Standard module files

| File | Purpose |
|------|---------|
| `main.tf` | Primary resources |
| `variables.tf` | All input variable declarations |
| `outputs.tf` | All output declarations |
| `locals.tf` | Local values (when there are several) |
| `providers.tf` | Provider configuration (root modules only) |
| `versions.tf` | `terraform {}` block with version constraints |
| `data.tf` | Data sources (when there are many) |
| `terraform.tfvars` | Variable values (root modules only, gitignored) |
| `*.auto.tfvars` | Auto-loaded variable files |

### When to split into multiple files

Split `main.tf` when it grows beyond ~200 lines. Name files by logical grouping:

```
module/
  main.tf            # Core resources (VPC, subnets)
  security.tf        # Security groups, NACLs
  iam.tf             # IAM roles and policies
  monitoring.tf      # CloudWatch, alarms
  variables.tf
  outputs.tf
```

### Naming rules for files

- Always `snake_case` with `.tf` extension
- Name by purpose, not by resource type (`networking.tf` not `aws_vpc.tf`)
- Keep `variables.tf` and `outputs.tf` as single files (don't split per-resource)

## Variable Naming

```hcl
# Descriptive, snake_case, with description and type
variable "vpc_cidr_block" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

# Boolean variables: use enable_/is_/has_ prefix
variable "enable_monitoring" {
  type        = bool
  description = "Whether to create CloudWatch monitoring resources"
  default     = true
}

# List/map variables: use plural names
variable "subnet_ids" {
  type        = list(string)
  description = "List of subnet IDs for the service"
}

variable "extra_tags" {
  type        = map(string)
  description = "Additional tags to apply to all resources"
  default     = {}
}
```

### Variable ordering in `variables.tf`

1. Required variables (no default) first
2. Optional variables (with default) after
3. Group related variables together
4. Every variable MUST have a `description`

## Output Naming

```hcl
# Pattern: {resource_type}_{attribute}
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}

# Sensitive outputs
output "database_password" {
  description = "Generated database password"
  value       = random_password.db.result
  sensitive   = true
}
```

### Output rules

- Always include `description`
- Mark sensitive outputs with `sensitive = true`
- Use consistent naming: `{thing}_{attribute}` (e.g., `cluster_endpoint`, `bucket_arn`)
- Expose only what consumers need

## Module Naming

### Directory naming

```
modules/
  networking/          # snake_case or kebab-case, descriptive
  ecs-service/         # Kebab-case also acceptable
  iam-role/
  s3-bucket/
```

### Module source references

```hcl
# Local modules
module "networking" {
  source = "./modules/networking"
}

# Registry modules — always pin version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

# Git modules — always pin to tag or commit
module "custom" {
  source = "git::https://github.com/org/module.git?ref=v1.2.0"
}
```

### Module instance naming

```hcl
# Descriptive, purpose-based
module "app_database" { }
module "worker_queue" { }

# Not generic
module "db" { }        # Too abbreviated
module "module1" { }   # Meaningless
```

## Tag Standards

### Required tags

Apply these to all taggable resources:

```hcl
locals {
  required_tags = {
    Environment = var.environment     # dev, staging, prod
    Project     = var.project_name    # Project/application name
    ManagedBy   = "terraform"         # Management tool
    Owner       = var.team            # Team or individual responsible
  }
}
```

### Tag application pattern

```hcl
# Apply via merge in every resource
resource "aws_instance" "web" {
  # ...
  tags = merge(local.required_tags, {
    Name = "${local.name_prefix}-web"
    Role = "web-server"
  })
}

# Or use default_tags in provider (AWS)
provider "aws" {
  region = var.region

  default_tags {
    tags = local.required_tags
  }
}
```

### Tag naming rules

| Rule | Example | Anti-pattern |
|------|---------|-------------|
| PascalCase keys | `Environment` | `environment`, `ENVIRONMENT` |
| Lowercase values | `prod`, `web-server` | `PROD`, `Web-Server` |
| No spaces in values | `web-server` | `web server` |
| Use `Name` tag for display names | `Name = "myapp-prod-web"` | Unnamed resources |
