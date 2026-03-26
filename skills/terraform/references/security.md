# Terraform Security Best Practices

## Table of Contents

- [Core Principles](#core-principles)
- [Secrets Management](#secrets-management)
- [Encryption](#encryption)
- [IAM Best Practices](#iam-best-practices)
- [Network Security](#network-security)
- [S3 Bucket Security](#s3-bucket-security)
- [Database Security](#database-security)
- [Compliance Scanning](#compliance-scanning)
- [Security Checklist](#security-checklist)

## Core Principles

1. **Never store secrets in code or state** — use secret managers, mark variables `sensitive`
2. **Encrypt everything at rest and in transit** — enable by default, not as an afterthought
3. **Least privilege** — grant only the minimum permissions required
4. **Defense in depth** — multiple layers of security (network + IAM + encryption)
5. **Audit and monitor** — enable logging, CloudTrail, access logs

## Secrets Management

### Never in Terraform code

```hcl
# NEVER do this
resource "aws_db_instance" "db" {
  password = "my-secret-password"  # Exposed in state file and VCS
}

# NEVER do this either
variable "db_password" {
  default = "my-secret-password"   # Still exposed in code
}
```

### Use secret managers

```hcl
# AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/password"
}

resource "aws_db_instance" "db" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}

# AWS SSM Parameter Store
data "aws_ssm_parameter" "api_key" {
  name            = "/app/prod/api-key"
  with_decryption = true
}
```

### Generate secrets with Terraform

```hcl
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret" "db_password" {
  name = "${var.project}-${var.environment}-db-password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db.result
}
```

### Mark sensitive variables and outputs

```hcl
variable "database_password" {
  type        = string
  description = "Database master password"
  sensitive   = true
}

output "database_password" {
  value     = random_password.db.result
  sensitive = true
}
```

### State file security

State files contain secrets in plaintext. Always:
- Use encrypted remote backends (S3 with SSE, GCS with CMEK)
- Restrict access to state bucket/container with IAM
- Enable versioning on state bucket for recovery
- Never commit state files to version control

## Encryption

### At rest — enable by default

```hcl
# S3 — server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.this.arn
    }
    bucket_key_enabled = true
  }
}

# RDS — storage encryption
resource "aws_db_instance" "db" {
  storage_encrypted = true
  kms_key_id        = aws_kms_key.db.arn
  # ...
}

# EBS — default encryption
resource "aws_ebs_encryption_by_default" "this" {
  enabled = true
}
```

### In transit — enforce TLS

```hcl
# ElastiCache — encryption in transit
resource "aws_elasticache_replication_group" "redis" {
  transit_encryption_enabled = true
  at_rest_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result
  # ...
}

# ALB — redirect HTTP to HTTPS
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.this.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

## IAM Best Practices

### Least privilege policies

```hcl
# Specific actions, specific resources — never use "*"
data "aws_iam_policy_document" "app" {
  statement {
    sid    = "ReadS3"
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:ListBucket",
    ]
    resources = [
      aws_s3_bucket.app.arn,
      "${aws_s3_bucket.app.arn}/*",
    ]
  }

  statement {
    sid    = "WriteSQS"
    effect = "Allow"
    actions = [
      "sqs:SendMessage",
      "sqs:GetQueueAttributes",
    ]
    resources = [
      aws_sqs_queue.tasks.arn,
    ]
  }
}
```

### Service roles (assume role pattern)

```hcl
# Trust policy for ECS tasks
data "aws_iam_policy_document" "ecs_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "task" {
  name               = "${var.name}-task-role"
  assume_role_policy = data.aws_iam_policy_document.ecs_assume.json
}

resource "aws_iam_role_policy" "task" {
  role   = aws_iam_role.task.id
  policy = data.aws_iam_policy_document.app.json
}
```

### Avoid anti-patterns

```hcl
# NEVER: wildcard actions and resources
statement {
  actions   = ["*"]
  resources = ["*"]
}

# NEVER: inline JSON policies (use aws_iam_policy_document)
resource "aws_iam_role_policy" "bad" {
  policy = jsonencode({
    Statement = [{ Action = "*", Resource = "*", Effect = "Allow" }]
  })
}

# NEVER: long-lived access keys in code
resource "aws_iam_access_key" "user" {
  user = aws_iam_user.deploy.name
  # These will be in state — use OIDC or assume role instead
}
```

## Network Security

### Security groups — explicit rules

```hcl
resource "aws_security_group" "web" {
  name        = "${var.name}-web"
  description = "Security group for web servers"
  vpc_id      = var.vpc_id

  # Explicit ingress rules — no 0.0.0.0/0 for SSH
  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Restrict egress when possible
  egress {
    description = "HTTPS to internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, { Name = "${var.name}-web" })
}
```

### Security group references (prefer over CIDRs)

```hcl
# Allow app servers to reach database via security group reference
resource "aws_security_group_rule" "db_from_app" {
  type                     = "ingress"
  security_group_id        = aws_security_group.database.id
  source_security_group_id = aws_security_group.app.id
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  description              = "PostgreSQL from app servers"
}
```

### Private subnets for sensitive resources

```hcl
# Databases, caches, internal services — always in private subnets
resource "aws_db_instance" "db" {
  db_subnet_group_name = aws_db_subnet_group.private.name
  publicly_accessible  = false
  # ...
}
```

## S3 Bucket Security

```hcl
# Block all public access (default for new buckets, but be explicit)
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Enable versioning
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

# Enable access logging
resource "aws_s3_bucket_logging" "this" {
  bucket        = aws_s3_bucket.this.id
  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "s3-access-logs/${aws_s3_bucket.this.id}/"
}
```

## Database Security

```hcl
resource "aws_db_instance" "db" {
  # Encryption
  storage_encrypted = true
  kms_key_id        = aws_kms_key.db.arn

  # Network isolation
  db_subnet_group_name   = aws_db_subnet_group.private.name
  publicly_accessible    = false
  vpc_security_group_ids = [aws_security_group.database.id]

  # Authentication
  manage_master_user_password = true  # AWS manages the password (Terraform 5.x+)

  # Backup and recovery
  backup_retention_period = 7
  deletion_protection     = true
  skip_final_snapshot     = false
  final_snapshot_identifier = "${var.name}-final-${formatdate("YYYY-MM-DD", timestamp())}"

  # Logging
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  # Maintenance
  auto_minor_version_upgrade = true
}
```

## Compliance Scanning

### Tools

| Tool | Focus | Usage |
|------|-------|-------|
| `terraform validate` | Syntax and internal consistency | Always — before plan |
| `tflint` | Provider-specific linting, best practices | Always — CI/CD |
| `checkov` | Security and compliance policies | Recommended — CI/CD |
| `tfsec` / `trivy` | Security scanning | Recommended — CI/CD |
| `OPA/Conftest` | Custom policy enforcement | For organizations with custom policies |

### Integration pattern

```bash
# Pre-commit or CI pipeline
terraform fmt -check -recursive
terraform validate
tflint --recursive
checkov -d . --framework terraform
```

## Security Checklist

### For every resource

- [ ] Encryption at rest enabled
- [ ] Encryption in transit enabled (where applicable)
- [ ] Least privilege IAM permissions
- [ ] No hardcoded secrets or credentials
- [ ] Sensitive variables marked `sensitive = true`
- [ ] Resources in appropriate network tier (public/private)

### For S3 buckets

- [ ] Public access blocked
- [ ] Versioning enabled
- [ ] Server-side encryption enabled
- [ ] Access logging enabled
- [ ] Lifecycle rules configured (if applicable)

### For databases

- [ ] Storage encryption enabled
- [ ] In private subnet, not publicly accessible
- [ ] Password managed by secret manager
- [ ] Backup retention configured
- [ ] Deletion protection enabled
- [ ] Audit logging enabled

### For IAM

- [ ] No wildcard actions or resources
- [ ] Using `aws_iam_policy_document` (not inline JSON)
- [ ] Service roles with proper trust policies
- [ ] No long-lived access keys (prefer OIDC/assume role)
- [ ] MFA enforced for human users

### For networking

- [ ] Security groups have descriptions on all rules
- [ ] No `0.0.0.0/0` on SSH/RDP ports
- [ ] Prefer security group references over CIDR blocks
- [ ] Sensitive resources in private subnets
- [ ] VPC flow logs enabled
