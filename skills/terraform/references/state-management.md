# Terraform State Management

## Table of Contents

- [Remote Backends](#remote-backends)
- [State Locking](#state-locking)
- [State Isolation](#state-isolation)
- [Workspaces](#workspaces)
- [State Operations](#state-operations)
- [Migration Patterns](#migration-patterns)

## Remote Backends

### Always use remote state for shared infrastructure

Local state is acceptable only for learning or truly single-operator projects.

### AWS S3 Backend (most common)

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### GCP GCS Backend

```hcl
terraform {
  backend "gcs" {
    bucket = "mycompany-terraform-state"
    prefix = "prod/networking"
  }
}
```

### Azure Blob Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "prod/networking/terraform.tfstate"
  }
}
```

### Terraform Cloud / HCP Terraform

```hcl
terraform {
  cloud {
    organization = "mycompany"
    workspaces {
      name = "prod-networking"
    }
  }
}
```

### Backend configuration best practices

- **Encrypt state** — always (S3 SSE, GCS CMEK, Azure encryption)
- **Restrict access** — IAM policies limiting who can read/write state
- **Enable versioning** — on the storage bucket for state recovery
- **Use consistent key naming** — `{environment}/{component}/terraform.tfstate`
- **Separate state bucket** — dedicated bucket for state, not shared with application data

## State Locking

### Why locking matters

Without locking, concurrent `terraform apply` runs can corrupt state or create duplicate resources.

### DynamoDB locking (AWS)

```hcl
# Create the lock table (do this once, manually or with a bootstrap config)
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name      = "terraform-locks"
    ManagedBy = "terraform"
  }
}
```

### Force-unlock (emergency only)

```bash
# Only when you're certain no other operation is running
terraform force-unlock LOCK_ID
```

Never force-unlock during a legitimate operation — it can corrupt state.

## State Isolation

### Strategy: separate state files per component

```
terraform-state-bucket/
  networking/terraform.tfstate       # VPC, subnets, route tables
  database/terraform.tfstate         # RDS, ElastiCache
  application/terraform.tfstate      # ECS, Lambda, API Gateway
  iam/terraform.tfstate              # IAM users, roles, policies
  monitoring/terraform.tfstate       # CloudWatch, alarms, dashboards
```

### Benefits of state isolation

| Benefit | Explanation |
|---------|-------------|
| **Blast radius** | A bad apply only affects one component |
| **Speed** | Smaller state = faster plan/apply |
| **Permissions** | Different teams can own different state files |
| **Concurrency** | Teams can work on different components simultaneously |
| **Recovery** | Easier to recover one component vs entire infrastructure |

### Cross-state references

```hcl
# In application/ — reference networking outputs
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_ecs_service" "app" {
  network_configuration {
    subnets         = data.terraform_remote_state.networking.outputs.private_subnet_ids
    security_groups = [aws_security_group.app.id]
  }
}
```

### Alternative: `terraform_remote_state` vs data sources

| Approach | When to Use |
|----------|-------------|
| `terraform_remote_state` | Same team, same Terraform setup |
| Data sources (e.g., `aws_vpc`, `aws_subnet`) | Cross-team, or when outputs aren't exposed |
| SSM Parameter Store / Consul | Service discovery, dynamic values |

## Workspaces

### When to use workspaces

Workspaces are useful for deploying the **same configuration** to multiple environments:

```bash
terraform workspace new staging
terraform workspace new prod
terraform workspace select prod
terraform workspace list
```

```hcl
# Use workspace name in configuration
locals {
  environment = terraform.workspace
  name_prefix = "myapp-${local.environment}"
}

resource "aws_instance" "web" {
  instance_type = local.environment == "prod" ? "m5.large" : "t3.micro"
  tags = {
    Environment = local.environment
  }
}
```

### When NOT to use workspaces

- Different environments need **significantly different configurations** — use separate directories
- Different environments are managed by **different teams** — separate state files with separate access
- You need **different backends per environment** — workspaces share the same backend
- The environment differences go beyond variable values

### Recommended pattern: directories over workspaces

For most teams, separate directories per environment are clearer and safer:

```
environments/
  dev/
    main.tf       # Different module versions, smaller instances
    backend.tf    # Separate state file
  staging/
    main.tf
    backend.tf
  prod/
    main.tf       # Pinned versions, larger instances, HA
    backend.tf    # Separate state file, stricter access
```

## State Operations

### Safe operations

```bash
# List resources in state
terraform state list

# Show a specific resource
terraform state show aws_instance.web

# Pull remote state locally (read-only inspection)
terraform state pull > state.json
```

### Dangerous operations (require caution)

```bash
# Move/rename a resource in state (prevents destroy/recreate)
terraform state mv aws_instance.old_name aws_instance.new_name

# Remove a resource from state (Terraform forgets it, resource still exists)
terraform state rm aws_instance.web

# Import an existing resource into state
terraform import aws_s3_bucket.existing my-existing-bucket
```

### Safety rules for state operations

1. **Always backup state first:** `terraform state pull > backup.tfstate`
2. **Run `terraform plan` after any state operation** to verify no unintended changes
3. **Never edit state JSON directly** — use `terraform state` commands
4. **Prefer `moved` blocks over `terraform state mv`** — moved blocks are declarative and reviewable
5. **Prefer `import` blocks over `terraform import`** — import blocks are declarative (Terraform 1.5+)

## Migration Patterns

### Moving from local to remote state

```bash
# 1. Add backend configuration to your code
# 2. Run init — Terraform will detect the change
terraform init
# Terraform will ask: "Do you want to copy existing state to the new backend?"
# Answer: yes
```

### Moving resources between state files

```bash
# 1. Pull source state
terraform state pull > source.tfstate

# 2. Remove from source
terraform state rm aws_instance.web

# 3. Import into destination
cd ../destination
terraform import aws_instance.web i-1234567890abcdef0

# 4. Verify both states
terraform plan  # Should show no changes in both source and destination
```

### Refactoring with `moved` blocks (preferred)

```hcl
# Renaming a resource
moved {
  from = aws_instance.web_server
  to   = aws_instance.web
}

# Moving into a module
moved {
  from = aws_s3_bucket.logs
  to   = module.logging.aws_s3_bucket.this
}
```

After applying the moved blocks and confirming `terraform plan` shows no changes, you can remove the moved blocks in a subsequent commit.

### State file recovery

If state is corrupted or lost:

1. **Check bucket versioning** — restore previous state version
2. **Use `terraform import`** to re-import resources one by one
3. **Never blindly apply** with missing state — it will try to create duplicates
