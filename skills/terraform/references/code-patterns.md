# Terraform Code Patterns

## Table of Contents

- [Resource Meta-Arguments](#resource-meta-arguments)
- [Dynamic Blocks](#dynamic-blocks)
- [Locals](#locals)
- [Data Sources](#data-sources)
- [Conditional Resources](#conditional-resources)
- [Variable Validation](#variable-validation)
- [Functions and Expressions](#functions-and-expressions)
- [Moved Blocks](#moved-blocks)
- [Import Blocks](#import-blocks)

## Resource Meta-Arguments

### `for_each` (preferred over `count`)

Use `for_each` for sets of similar resources. It creates stable addressing by key, avoiding index-shift issues.

```hcl
# Map-based for_each
resource "aws_security_group_rule" "ingress" {
  for_each = {
    http  = { port = 80,  cidr = ["0.0.0.0/0"] }
    https = { port = 443, cidr = ["0.0.0.0/0"] }
    ssh   = { port = 22,  cidr = ["10.0.0.0/8"] }
  }

  type              = "ingress"
  security_group_id = aws_security_group.web.id
  from_port         = each.value.port
  to_port           = each.value.port
  protocol          = "tcp"
  cidr_blocks       = each.value.cidr
}

# Set-based for_each
resource "aws_iam_user" "developers" {
  for_each = toset(var.developer_names)
  name     = each.value
}
```

### `count` (use only for simple on/off)

```hcl
resource "aws_cloudwatch_log_group" "app" {
  count = var.enable_logging ? 1 : 0
  name  = "/app/${var.name}"
}
```

### `depends_on` (use sparingly)

Only when Terraform cannot infer the dependency automatically:

```hcl
resource "aws_ecs_service" "app" {
  # ...
  depends_on = [aws_iam_role_policy_attachment.ecs_task]
}
```

### `lifecycle`

```hcl
resource "aws_instance" "web" {
  # ...

  lifecycle {
    create_before_destroy = true              # Zero-downtime replacement
    prevent_destroy       = true              # Protect critical resources
    ignore_changes        = [tags["UpdatedBy"]] # Ignore external changes
  }
}
```

## Dynamic Blocks

Use for repeated nested blocks driven by data:

```hcl
resource "aws_security_group" "web" {
  name   = "${var.name}-sg"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Avoid nesting dynamic blocks more than one level deep — it becomes unreadable.

## Locals

Use locals to reduce repetition and improve readability:

```hcl
locals {
  # Common tags applied to all resources
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
    Owner       = var.team
  }

  # Computed values
  name_prefix = "${var.project_name}-${var.environment}"
  is_prod     = var.environment == "prod"
}

resource "aws_instance" "web" {
  instance_type = local.is_prod ? "m5.large" : "t3.micro"
  tags          = merge(local.common_tags, { Name = "${local.name_prefix}-web" })
}
```

## Data Sources

Use data sources to reference existing infrastructure:

```hcl
# Look up existing resources
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["main-vpc"]
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# Reference current account/region
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
}
```

## Conditional Resources

### Feature flags pattern

```hcl
variable "features" {
  type = object({
    enable_monitoring = bool
    enable_backups    = bool
    enable_waf        = bool
  })
  default = {
    enable_monitoring = true
    enable_backups    = true
    enable_waf        = false
  }
}

resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = var.features.enable_monitoring ? 1 : 0
  # ...
}

resource "aws_backup_plan" "daily" {
  count = var.features.enable_backups ? 1 : 0
  # ...
}
```

### Referencing conditional resources

```hcl
# Safe reference to conditional resource
output "alarm_arn" {
  value = var.features.enable_monitoring ? aws_cloudwatch_metric_alarm.cpu[0].arn : null
}

# With try() for cleaner syntax
output "alarm_arn" {
  value = try(aws_cloudwatch_metric_alarm.cpu[0].arn, null)
}
```

## Variable Validation

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"

  validation {
    condition     = can(regex("^(t3|t4g|m5|m6i|c5|c6i)\\.", var.instance_type))
    error_message = "Instance type must be t3, t4g, m5, m6i, c5, or c6i family."
  }
}

variable "cidr_block" {
  type        = string
  description = "VPC CIDR block"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "port" {
  type        = number
  description = "Application port"

  validation {
    condition     = var.port > 0 && var.port <= 65535
    error_message = "Port must be between 1 and 65535."
  }
}
```

## Functions and Expressions

### Common patterns

```hcl
locals {
  # Merge maps with precedence (right wins)
  tags = merge(var.default_tags, var.extra_tags, { Name = var.name })

  # Flatten nested structures
  subnet_pairs = flatten([
    for az in var.availability_zones : [
      for type in ["public", "private"] : {
        az   = az
        type = type
        cidr = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, az) * 2 + (type == "private" ? 1 : 0))
      }
    ]
  ])

  # Filter and transform
  prod_instances = { for k, v in var.instances : k => v if v.environment == "prod" }

  # Lookup with default
  instance_type = lookup(var.instance_types, var.environment, "t3.micro")

  # Coalesce — first non-null/empty value
  name = coalesce(var.custom_name, "${var.project}-${var.environment}")

  # String templates
  user_data = templatefile("${path.module}/templates/init.sh", {
    environment = var.environment
    region      = var.region
  })
}
```

## Moved Blocks

Refactor without destroying/recreating resources:

```hcl
# Rename a resource
moved {
  from = aws_instance.web_server
  to   = aws_instance.web
}

# Move into a module
moved {
  from = aws_s3_bucket.logs
  to   = module.logging.aws_s3_bucket.this
}

# Move within for_each
moved {
  from = aws_iam_user.users["old_key"]
  to   = aws_iam_user.users["new_key"]
}
```

Run `terraform plan` after adding moved blocks to verify no destroy/create operations.

## Import Blocks

Bring existing resources under Terraform management (Terraform 1.5+):

```hcl
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}

resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
  # Write configuration to match existing resource
}
```

Run `terraform plan` after importing to verify the configuration matches. Fix any drift before applying.
