# Terraform Module Best Practices

## Table of Contents

- [When to Create a Module](#when-to-create-a-module)
- [Standard Module Structure](#standard-module-structure)
- [Input Design](#input-design)
- [Output Design](#output-design)
- [Composition Patterns](#composition-patterns)
- [Registry Modules](#registry-modules)
- [Versioning](#versioning)

## When to Create a Module

### Create a module when:

- Resources are **reused across environments or projects** (e.g., ECS service, S3 bucket with standard config)
- A logical group of resources **always deploy together** (e.g., RDS + subnet group + security group + parameter group)
- You want to **enforce standards** (e.g., S3 buckets always have encryption and versioning)
- The root module exceeds **~300 lines** and needs logical decomposition

### Don't create a module when:

- Resources are **used once** and inline config is clear enough
- The module would be a **thin wrapper** around a single resource with no added logic
- You're abstracting only to **reduce line count** (not to add reuse or enforce patterns)
- It only exists because "modules are best practice" — modules add indirection cost

### Decision tree

```
Is this resource pattern used in 2+ places?
  ├── Yes → Create a module
  └── No → Does it enforce important standards?
        ├── Yes → Create a module
        └── No → Is the root module getting too large (>300 lines)?
              ├── Yes → Consider splitting into child modules by logical grouping
              └── No → Keep it inline
```

## Standard Module Structure

```
modules/my-module/
  main.tf           # Resources
  variables.tf      # Inputs (all with descriptions and types)
  outputs.tf        # Outputs (all with descriptions)
  versions.tf       # Required terraform and provider versions
  locals.tf         # Computed values (optional)
  data.tf           # Data sources (optional, or inline in main.tf)
  README.md         # Usage documentation
```

### Root module structure (environments)

```
environments/
  prod/
    main.tf         # Module calls with prod values
    providers.tf    # Provider config with prod backend
    variables.tf
    outputs.tf
    terraform.tfvars
    versions.tf
  staging/
    main.tf
    providers.tf
    variables.tf
    outputs.tf
    terraform.tfvars
    versions.tf
modules/
  networking/
  compute/
  database/
```

## Input Design

### Principles

1. **Minimal required inputs** — sensible defaults for everything else
2. **Type constraints** — always specify `type`
3. **Validation** — validate inputs where constraints exist
4. **Descriptions** — every variable gets a `description`

### Pattern: simple inputs with smart defaults

```hcl
variable "name" {
  type        = string
  description = "Name prefix for all resources in this module"
}

variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, prod)"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

variable "enable_monitoring" {
  type        = bool
  description = "Enable CloudWatch detailed monitoring"
  default     = true
}
```

### Pattern: object variables for related config

```hcl
variable "database" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    allocated_storage = number
    multi_az       = optional(bool, false)
    backup_retention = optional(number, 7)
  })
  description = "Database configuration"
}

# Usage in module call
module "database" {
  source = "./modules/database"

  database = {
    engine            = "postgres"
    engine_version    = "16.1"
    instance_class    = "db.t3.medium"
    allocated_storage = 50
    multi_az          = true
  }
}
```

### Anti-patterns

```hcl
# Too many individual variables for related config
variable "db_engine" { }
variable "db_version" { }
variable "db_instance_class" { }
variable "db_storage" { }
variable "db_multi_az" { }
# Better: group into object variable (see above)

# Passing through everything
variable "vpc_config" {
  type = any  # Avoid 'any' — loses type safety
}
```

## Output Design

### Principles

1. **Expose what consumers need** — don't output everything
2. **Consistent naming** — `{resource}_{attribute}` pattern
3. **Always describe** — every output gets a `description`
4. **Mark sensitive** — use `sensitive = true` for secrets

```hcl
output "cluster_id" {
  description = "ID of the ECS cluster"
  value       = aws_ecs_cluster.main.id
}

output "cluster_arn" {
  description = "ARN of the ECS cluster"
  value       = aws_ecs_cluster.main.arn
}

output "service_security_group_id" {
  description = "Security group ID for ECS services to use"
  value       = aws_security_group.ecs_service.id
}
```

## Composition Patterns

### Flat composition (preferred for most cases)

Root module calls child modules directly:

```hcl
module "networking" {
  source      = "./modules/networking"
  environment = var.environment
  vpc_cidr    = var.vpc_cidr
}

module "database" {
  source     = "./modules/database"
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
}

module "application" {
  source       = "./modules/application"
  vpc_id       = module.networking.vpc_id
  subnet_ids   = module.networking.private_subnet_ids
  database_url = module.database.connection_url
}
```

### Nested composition (use sparingly)

Module calls other modules. Only when sub-modules are implementation details:

```hcl
# modules/application/main.tf
module "ecs_service" {
  source = "../ecs-service"
  # ...
}

module "alb" {
  source = "../alb"
  # ...
}
```

Avoid nesting deeper than 2 levels — it becomes hard to debug.

### Cross-state composition

Use `terraform_remote_state` or data sources for cross-state references:

```hcl
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

module "application" {
  source     = "./modules/application"
  vpc_id     = data.terraform_remote_state.networking.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
}
```

## Registry Modules

### Using community modules

```hcl
# Always pin version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project}-${var.environment}"
  cidr = var.vpc_cidr

  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"

  tags = local.common_tags
}
```

### When to use registry vs custom modules

| Use Registry Module | Use Custom Module |
|---------------------|-------------------|
| Standard patterns (VPC, ECS, RDS) | Organization-specific logic |
| Well-maintained, widely used | Tight integration requirements |
| Saves significant development time | Need full control over resources |
| Acceptable to depend on external code | Security/compliance restrictions |

## Versioning

### Version constraint syntax

```hcl
version = "5.0.0"     # Exact version — most restrictive
version = "~> 5.0"    # >= 5.0.0, < 6.0.0 — recommended for most cases
version = ">= 5.0"    # Any version >= 5.0.0 — too permissive, avoid
```

### Pinning strategy

| Context | Strategy |
|---------|----------|
| Production root modules | Pin exact or use `~>` minor |
| Shared/reusable modules | Use `~>` to allow patch updates |
| Development/experimentation | `~>` major is acceptable |
| CI/CD pipelines | Lock file (`terraform.lock.hcl`) committed |

### Lock file

Always commit `terraform.lock.hcl` to version control. It ensures reproducible builds:

```bash
# Update lock file after changing versions
terraform init -upgrade

# Verify lock file is up to date
terraform init
```
