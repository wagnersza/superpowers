# AWS CLI Common Patterns

## Table of Contents

- [Multi-Resource Discovery](#multi-resource-discovery)
- [Cross-Service Lookups](#cross-service-lookups)
- [Cost and Billing](#cost-and-billing)
- [Troubleshooting Patterns](#troubleshooting-patterns)
- [Batch Operations](#batch-operations)
- [Output Formatting](#output-formatting)

## Multi-Resource Discovery

### Full VPC inventory

When you need a complete picture of a VPC's resources:

```bash
VPC_ID="vpc-xxxxx"

# 1. VPC details
aws ec2 describe-vpcs --vpc-ids $VPC_ID --output json

# 2. Subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# 3. Security groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" --query 'SecurityGroups[].{ID:GroupId,Name:GroupName}' --output table

# 4. NAT Gateways
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" --query 'NatGateways[].{ID:NatGatewayId,SubnetId:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp}' --output table

# 5. Internet Gateways
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query 'InternetGateways[].InternetGatewayId' --output text

# 6. Route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query 'RouteTables[].{ID:RouteTableId,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# 7. VPC Endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" --query 'VpcEndpoints[].{ID:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State}' --output table
```

### ECS service full picture

```bash
CLUSTER="my-cluster"
SERVICE="my-service"

# Service overview
aws ecs describe-services --cluster $CLUSTER --services $SERVICE --query 'services[0].{Status:status,Desired:desiredCount,Running:runningCount,TaskDef:taskDefinition,LB:loadBalancers}' --output json

# Current task definition
TASK_DEF=$(aws ecs describe-services --cluster $CLUSTER --services $SERVICE --query 'services[0].taskDefinition' --output text)
aws ecs describe-task-definition --task-definition $TASK_DEF --query 'taskDefinition.{Family:family,CPU:cpu,Memory:memory,Containers:containerDefinitions[].{Name:name,Image:image,Ports:portMappings[].containerPort,Env:environment[].name}}' --output json

# Running tasks
aws ecs list-tasks --cluster $CLUSTER --service-name $SERVICE --desired-status RUNNING --query 'taskArns' --output text

# Service events (recent deployments/issues)
aws ecs describe-services --cluster $CLUSTER --services $SERVICE --query 'services[0].events[:5].{Time:createdAt,Message:message}' --output table
```

### RDS full inspection

```bash
DB_ID="my-database"

# Instance details
aws rds describe-db-instances --db-instance-identifier $DB_ID --query 'DBInstances[0].{Engine:Engine,Version:EngineVersion,Class:DBInstanceClass,Storage:AllocatedStorage,IOPS:Iops,MultiAZ:MultiAZ,Encrypted:StorageEncrypted,Endpoint:Endpoint.Address,Port:Endpoint.Port,VPC:DBSubnetGroup.VpcId,SecurityGroups:VpcSecurityGroups[].VpcSecurityGroupId,BackupRetention:BackupRetentionPeriod,DeletionProtection:DeletionProtection}' --output json

# Parameter group
aws rds describe-db-instances --db-instance-identifier $DB_ID --query 'DBInstances[0].DBParameterGroups[].DBParameterGroupName' --output text

# Recent events
aws rds describe-events --source-identifier $DB_ID --source-type db-instance --duration 1440 --query 'Events[].{Time:Date,Message:Message}' --output table
```

## Cross-Service Lookups

### Find what uses a security group

```bash
SG_ID="sg-xxxxx"

# EC2 instances
aws ec2 describe-instances --filters "Name=instance.group-id,Values=$SG_ID" --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# ENIs (catches Lambda, ECS, RDS, etc.)
aws ec2 describe-network-interfaces --filters "Name=group-id,Values=$SG_ID" --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Type:InterfaceType,Description:Description,AZ:AvailabilityZone}' --output table

# RDS instances
aws rds describe-db-instances --query "DBInstances[?VpcSecurityGroups[?VpcSecurityGroupId=='$SG_ID']].DBInstanceIdentifier" --output table
```

### Find what uses a subnet

```bash
SUBNET_ID="subnet-xxxxx"

# EC2 instances
aws ec2 describe-instances --filters "Name=subnet-id,Values=$SUBNET_ID" --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# ENIs
aws ec2 describe-network-interfaces --filters "Name=subnet-id,Values=$SUBNET_ID" --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Type:InterfaceType,Description:Description}' --output table

# RDS subnet groups containing this subnet
aws rds describe-db-subnet-groups --query "DBSubnetGroups[?Subnets[?SubnetIdentifier=='$SUBNET_ID']].DBSubnetGroupName" --output table
```

### Find IAM role consumers

```bash
ROLE_NAME="my-role"

# ECS task definitions using this role
aws ecs list-task-definitions --query 'taskDefinitionArns' --output text | tr '\t' '\n' | while read td; do
  aws ecs describe-task-definition --task-definition "$td" --query "taskDefinition.{Family:family,TaskRole:taskRoleArn,ExecRole:executionRoleArn}" --output text | grep -q "$ROLE_NAME" && echo "$td"
done

# Lambda functions using this role
ROLE_ARN=$(aws iam get-role --role-name $ROLE_NAME --query 'Role.Arn' --output text)
aws lambda list-functions --query "Functions[?Role=='$ROLE_ARN'].FunctionName" --output table

# EC2 instance profiles
aws iam list-instance-profiles-for-role --role-name $ROLE_NAME --query 'InstanceProfiles[].InstanceProfileName' --output table
```

## Cost and Billing

```bash
# Current month costs by service
aws ce get-cost-and-usage \
  --time-period Start=$(date -u +%Y-%m-01),End=$(date -u +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[?Metrics.BlendedCost.Amount>`0`] | sort_by(@, &to_number(Metrics.BlendedCost.Amount)) | reverse(@) | [:10].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}' \
  --output table

# Specific resource costs (requires Cost Explorer tags)
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --filter '{"Tags":{"Key":"Project","Values":["myapp"]}}' \
  --query 'ResultsByTime[0].Total.BlendedCost.{Amount:Amount,Unit:Unit}'
```

## Troubleshooting Patterns

### ECS service not starting

```bash
CLUSTER="my-cluster"
SERVICE="my-service"

# Check service events for errors
aws ecs describe-services --cluster $CLUSTER --services $SERVICE --query 'services[0].events[:10].{Time:createdAt,Message:message}' --output table

# Check stopped tasks for error reasons
TASKS=$(aws ecs list-tasks --cluster $CLUSTER --service-name $SERVICE --desired-status STOPPED --query 'taskArns[:3]' --output text)
aws ecs describe-tasks --cluster $CLUSTER --tasks $TASKS --query 'tasks[].{StopCode:stopCode,StopReason:stoppedReason,Containers:containers[].{Name:name,ExitCode:exitCode,Reason:reason}}' --output json
```

### Security group connectivity check

```bash
# Check if SG allows specific port
aws ec2 describe-security-group-rules --filters "Name=group-id,Values=sg-xxxxx" --query "SecurityGroupRules[?FromPort<=\`443\` && ToPort>=\`443\` && !IsEgress].{CIDR:CidrIpv4,SourceSG:ReferencedGroupInfo.GroupId,Description:Description}" --output table
```

### DNS resolution check

```bash
# Check Route53 record
aws route53 list-resource-record-sets --hosted-zone-id ZONE_ID --query "ResourceRecordSets[?Name=='myapp.example.com.']" --output json
```

## Batch Operations

### Tag all untagged resources

```bash
# Find untagged EC2 instances
aws ec2 describe-instances --query 'Reservations[].Instances[?!Tags].InstanceId' --output text

# Get all resources with a specific tag
aws resourcegroupstaggingapi get-resources --tag-filters Key=Environment,Values=prod --query 'ResourceTagMappingList[].ResourceARN' --output text
```

### Multi-region queries

```bash
# Check a resource across all regions
for region in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  echo "=== $region ==="
  aws ec2 describe-vpcs --region $region --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock}' --output table 2>/dev/null
done
```

## Output Formatting

### JMESPath essentials

```bash
# Select fields
--query 'Items[].{ID:Id,Name:Name}'

# Filter results
--query 'Items[?Status==`active`]'

# Nested access
--query 'Items[].Tags[?Key==`Name`].Value | []'

# Sort
--query 'sort_by(Items, &Name)'

# First N results
--query 'Items[:5]'

# Join values
--query 'Items[].Name | join(`, `, @)'

# Conditional/coalesce
--query 'Items[].{Name:Name||`unnamed`}'

# Count
--query 'length(Items)'

# Flatten nested arrays
--query 'Items[].SubItems[]'
```

### Output formats

| Format | When to Use |
|--------|-------------|
| `--output table` | Human-readable overview, scanning resources |
| `--output json` | Detailed inspection, programmatic processing |
| `--output text` | Piping to other commands, scripting |
| `--output yaml` | Readable structured data (AWS CLI v2) |
