---
name: aws-cli
description: "Use when needing to query, verify, or discover AWS infrastructure state using the AWS CLI. Use when: (1) Checking current state of AWS resources before making changes, (2) Verifying resource existence, configuration, or dependencies, (3) Discovering infrastructure details like VPC IDs, subnet CIDRs, security groups, IAM roles, (4) Confirming changes after terraform apply or manual modifications, (5) Debugging AWS resource issues or connectivity problems, (6) Listing resources across accounts or regions, (7) Any task requiring real-time AWS infrastructure information"
---

# AWS CLI Infrastructure Discovery

Use AWS CLI to **query real infrastructure state** before making assumptions. Never guess resource IDs, configurations, ARNs, or relationships — always verify with AWS CLI first.

## Critical Rules

### 1. Read-only — NEVER modify AWS resources directly (NON-NEGOTIABLE)

**AWS CLI is for READING infrastructure state only.** Never use AWS CLI to create, modify, or delete resources. All infrastructure changes MUST go through Terraform (or other IaC tools).

**FORBIDDEN commands — never execute these:**
- `aws * create-*`, `aws * update-*`, `aws * modify-*`, `aws * delete-*`, `aws * put-*`
- `aws * terminate-*`, `aws * remove-*`, `aws * detach-*`, `aws * deregister-*`
- `aws * start-*`, `aws * stop-*`, `aws * reboot-*` (for resources)
- `aws s3 rm`, `aws s3 cp`, `aws s3 mv`, `aws s3 sync` (to modify bucket contents)
- Any command that changes infrastructure state

**ALLOWED commands — read-only operations:**
- `aws * describe-*`, `aws * list-*`, `aws * get-*`
- `aws sts get-caller-identity`
- `aws ce get-cost-and-usage`
- `aws logs filter-log-events`, `aws logs get-log-events`
- `aws s3 ls`, `aws s3api head-object`
- Any command that only reads state

**If you need to change infrastructure:** Write Terraform code and follow the superpowers:terraform workflow (plan → handoff to user → user applies).

### 2. Never run destructive commands — human applies (NON-NEGOTIABLE)

**NEVER execute `terraform apply`, `terraform destroy`, or any command that modifies live infrastructure.** These commands must always be run by the human operator.

When changes are needed:
1. Write the Terraform code
2. Run `terraform plan` to show what will change
3. Present the plan and the exact apply command to the user
4. **The user runs the apply command themselves**

This applies to ALL destructive or state-modifying operations:
- `terraform apply` / `terraform destroy`
- `terraform state rm` / `terraform state mv` (present commands, let user run them)
- `terraform import` (present commands, let user run them)
- Any AWS CLI write command (see Rule #1)

### 3. Never assume — always verify

**NEVER assume any detail about existing infrastructure.** Before writing Terraform, CloudFormation, CDK, or any IaC code that references existing resources, verify with AWS CLI:

```bash
# Before referencing a VPC
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=*prod*" --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# Before referencing subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxxx" --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# Before referencing security groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-xxxxx" --query 'SecurityGroups[].{ID:GroupId,Name:GroupName,Description:Description}' --output table
```

### 4. Use structured output

Always use `--query` (JMESPath) and `--output table|json` for readable, parseable results:

```bash
# Table for human-readable overview
aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# JSON for detailed inspection
aws ec2 describe-instances --instance-ids i-1234567890abcdef0 --output json
```

### 5. Filter before fetching

Use server-side `--filters` to reduce response size, not client-side `jq`:

```bash
# Good: server-side filter
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" "Name=tag:Environment,Values=prod"

# Avoid: fetching everything and filtering locally
aws ec2 describe-instances | jq '.Reservations[].Instances[] | select(.State.Name == "running")'
```

### 6. Always specify region when ambiguous

```bash
# Explicit region
aws ec2 describe-vpcs --region us-east-1

# Or check current default
aws configure get region
```

## Discovery Workflow

When you need information about existing AWS infrastructure, follow this sequence:

1. **Identify what you need** — Which resource types, which account, which region?
2. **Query with AWS CLI** — Use the appropriate `describe-*`, `list-*`, or `get-*` command
3. **Extract relevant details** — Use `--query` to pull exactly what's needed
4. **Present findings** — Show the user what you found before proceeding
5. **Use verified values** — Only use resource IDs, ARNs, and configs that came from CLI output

## Quick Reference by Service

### EC2 / VPC / Networking

```bash
# VPCs
aws ec2 describe-vpcs --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# Subnets (by VPC)
aws ec2 describe-subnets --filters "Name=vpc-id,Values=VPC_ID" --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# Security groups (by VPC)
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=VPC_ID" --query 'SecurityGroups[].{ID:GroupId,Name:GroupName,Description:Description}' --output table

# Security group rules (specific group)
aws ec2 describe-security-group-rules --filters "Name=group-id,Values=SG_ID" --query 'SecurityGroupRules[].{Direction:IsEgress,Protocol:IpProtocol,FromPort:FromPort,ToPort:ToPort,CIDR:CidrIpv4,SourceSG:ReferencedGroupInfo.GroupId}' --output table

# NAT Gateways
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=VPC_ID" --query 'NatGateways[].{ID:NatGatewayId,State:State,SubnetId:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp}' --output table

# Route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=VPC_ID" --query 'RouteTables[].{ID:RouteTableId,Name:Tags[?Key==`Name`].Value|[0],Routes:Routes[].{Dest:DestinationCidrBlock,Target:GatewayId||NatGatewayId||TransitGatewayId}}' --output json

# Instances
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,AZ:Placement.AvailabilityZone,PrivateIP:PrivateIpAddress,Name:Tags[?Key==`Name`].Value|[0]}' --output table
```

### ECS

```bash
# Clusters
aws ecs list-clusters --query 'clusterArns' --output table

# Services in a cluster
aws ecs list-services --cluster CLUSTER_NAME --query 'serviceArns' --output table

# Service details
aws ecs describe-services --cluster CLUSTER_NAME --services SERVICE_NAME --query 'services[].{Name:serviceName,Status:status,Desired:desiredCount,Running:runningCount,TaskDef:taskDefinition}' --output table

# Task definition
aws ecs describe-task-definition --task-definition TASK_DEF_ARN --query 'taskDefinition.{Family:family,CPU:cpu,Memory:memory,Containers:containerDefinitions[].{Name:name,Image:image,CPU:cpu,Memory:memory}}' --output json
```

### RDS

```bash
# DB instances
aws rds describe-db-instances --query 'DBInstances[].{ID:DBInstanceIdentifier,Engine:Engine,Version:EngineVersion,Class:DBInstanceClass,Status:DBInstanceStatus,MultiAZ:MultiAZ,Storage:AllocatedStorage}' --output table

# Specific instance
aws rds describe-db-instances --db-instance-identifier INSTANCE_ID --query 'DBInstances[0].{Endpoint:Endpoint.Address,Port:Endpoint.Port,VPC:DBSubnetGroup.VpcId,Encrypted:StorageEncrypted,BackupRetention:BackupRetentionPeriod}' --output table

# DB clusters (Aurora)
aws rds describe-db-clusters --query 'DBClusters[].{ID:DBClusterIdentifier,Engine:Engine,Status:Status,Endpoint:Endpoint,ReaderEndpoint:ReaderEndpoint}' --output table
```

### S3

```bash
# List buckets
aws s3api list-buckets --query 'Buckets[].{Name:Name,Created:CreationDate}' --output table

# Bucket details
aws s3api get-bucket-versioning --bucket BUCKET_NAME
aws s3api get-bucket-encryption --bucket BUCKET_NAME
aws s3api get-public-access-block --bucket BUCKET_NAME
aws s3api get-bucket-location --bucket BUCKET_NAME
```

### IAM

```bash
# Roles
aws iam list-roles --query 'Roles[?contains(RoleName,`SEARCH_TERM`)].{Name:RoleName,ARN:Arn,Created:CreateDate}' --output table

# Role details and policies
aws iam get-role --role-name ROLE_NAME
aws iam list-attached-role-policies --role-name ROLE_NAME --query 'AttachedPolicies[].{Name:PolicyName,ARN:PolicyArn}' --output table
aws iam list-role-policies --role-name ROLE_NAME

# Users
aws iam list-users --query 'Users[].{Name:UserName,ARN:Arn,LastLogin:PasswordLastUsed}' --output table

# Policies
aws iam get-policy --policy-arn POLICY_ARN
aws iam get-policy-version --policy-arn POLICY_ARN --version-id v1 --query 'PolicyVersion.Document'
```

### EKS

```bash
# Clusters
aws eks list-clusters --query 'clusters' --output table

# Cluster details
aws eks describe-cluster --name CLUSTER_NAME --query 'cluster.{Name:name,Version:version,Status:status,Endpoint:endpoint,VPC:resourcesVpcConfig.vpcId,Subnets:resourcesVpcConfig.subnetIds}' --output json

# Node groups
aws eks list-nodegroups --cluster-name CLUSTER_NAME
aws eks describe-nodegroup --cluster-name CLUSTER_NAME --nodegroup-name NODEGROUP_NAME
```

### ElastiCache

```bash
# Redis/Memcached clusters
aws elasticache describe-cache-clusters --query 'CacheClusters[].{ID:CacheClusterId,Engine:Engine,Type:CacheNodeType,Status:CacheClusterStatus,Nodes:NumCacheNodes}' --output table

# Replication groups (Redis)
aws elasticache describe-replication-groups --query 'ReplicationGroups[].{ID:ReplicationGroupId,Status:Status,Encrypted:AtRestEncryptionEnabled,TransitEncryption:TransitEncryptionEnabled}' --output table
```

### Route53

```bash
# Hosted zones
aws route53 list-hosted-zones --query 'HostedZones[].{ID:Id,Name:Name,Private:Config.PrivateZone,Records:ResourceRecordSetCount}' --output table

# Records in a zone
aws route53 list-resource-record-sets --hosted-zone-id ZONE_ID --query 'ResourceRecordSets[].{Name:Name,Type:Type,TTL:TTL,Values:ResourceRecords[].Value|join(`, `,@)}' --output table
```

### CloudFront

```bash
# Distributions
aws cloudfront list-distributions --query 'DistributionList.Items[].{ID:Id,Domain:DomainName,Aliases:Aliases.Items|join(`, `,@),Status:Status}' --output table
```

### Lambda

```bash
# Functions
aws lambda list-functions --query 'Functions[].{Name:FunctionName,Runtime:Runtime,Memory:MemorySize,Timeout:Timeout,LastModified:LastModified}' --output table

# Function details
aws lambda get-function --function-name FUNCTION_NAME --query '{Runtime:Configuration.Runtime,Handler:Configuration.Handler,Role:Configuration.Role,VPC:Configuration.VpcConfig.VpcId}'
```

### SQS / SNS

```bash
# SQS queues
aws sqs list-queues --query 'QueueUrls' --output table

# Queue attributes
aws sqs get-queue-attributes --queue-url QUEUE_URL --attribute-names All

# SNS topics
aws sns list-topics --query 'Topics[].TopicArn' --output table
```

### Secrets Manager / SSM

```bash
# Secrets (names only, not values)
aws secretsmanager list-secrets --query 'SecretList[].{Name:Name,Description:Description,LastChanged:LastChangedDate}' --output table

# SSM parameters
aws ssm describe-parameters --query 'Parameters[].{Name:Name,Type:Type,LastModified:LastModifiedDate}' --output table

# Get parameter value (careful with sensitive values)
aws ssm get-parameter --name PARAM_NAME --with-decryption --query 'Parameter.{Name:Name,Value:Value,Type:Type}'
```

### CloudWatch

```bash
# Log groups
aws logs describe-log-groups --query 'logGroups[].{Name:logGroupName,Retention:retentionInDays,StoredBytes:storedBytes}' --output table

# Alarms
aws cloudwatch describe-alarms --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Metric:MetricName,Namespace:Namespace}' --output table
```

### Account / Caller Identity

```bash
# Who am I?
aws sts get-caller-identity --query '{Account:Account,ARN:Arn,UserId:UserId}' --output table

# Available regions
aws ec2 describe-regions --query 'Regions[].RegionName' --output table

# Current region
aws configure get region
```

## Reference Documentation

- **[references/common-patterns.md](references/common-patterns.md)** — Advanced query patterns, multi-resource discovery workflows, cross-service lookups, and troubleshooting techniques.
