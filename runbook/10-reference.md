# Module 10: Reference

Quick-reference material for daily use. Bookmark this page.

---

## 10.1 Complete FIS Actions Cheat Sheet

### Compute Actions

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:ec2:stop-instances` | Stop EC2 instances | `startInstancesAfterDuration` (ISO 8601) |
| `aws:ec2:terminate-instances` | Terminate EC2 instances | -- (irreversible) |
| `aws:ec2:reboot-instances` | Reboot EC2 instances | -- |
| `aws:ec2:send-spot-instance-interruptions` | Send 2-min Spot interruption notice | `durationBeforeInterruption` |
| `aws:ec2:api-insufficient-instance-capacity-error` | Inject InsufficientInstanceCapacity on EC2 API | `duration`, `percentage`, `availabilityZoneIdentifiers` |
| `aws:ec2:asg-insufficient-instance-capacity-error` | Inject capacity errors on ASG scale-out | `duration`, `percentage`, `availabilityZoneIdentifiers` |

### Network Actions

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:network:disrupt-connectivity` | Block traffic via NACLs | `duration`, `scope` (all/availability-zone/vpc/s3/dynamodb/prefix-list) |
| `aws:network:route-table-disrupt-cross-region-connectivity` | Block cross-region traffic from subnets | `duration` |
| `aws:network:transit-gateway-disrupt-cross-region-connectivity` | Block cross-region via TGW | `duration` |
| `aws:network:disrupt-vpc-endpoint` | Block interface VPC endpoint traffic | `duration` |

### Database Actions

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:rds:failover-db-cluster` | Failover Aurora/DocumentDB cluster | -- |
| `aws:rds:reboot-db-instances` | Reboot RDS instance | `forceFailover` (true/false) |

### Container Actions (ECS)

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:ecs:drain-container-instances` | Drain container instances | `duration`, `drainagePercentage` |
| `aws:ecs:stop-task` | Stop ECS tasks | -- |
| `aws:ecs:task-cpu-stress` | CPU stress on tasks | `duration`, `percent` |
| `aws:ecs:task-io-stress` | I/O stress on tasks | `duration`, `percent` |
| `aws:ecs:task-kill-process` | Kill process in task | `processName`, `signal` |
| `aws:ecs:task-network-blackhole-port` | Blackhole port traffic | `duration`, `port`, `protocol`, `trafficType` |
| `aws:ecs:task-network-latency` | Inject latency | `duration`, `delayMilliseconds`, `jitterMilliseconds` |
| `aws:ecs:task-network-packet-loss` | Inject packet loss | `duration`, `lossPercent` |

### Container Actions (EKS)

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:eks:pod-delete` | Delete pods | `kubernetesServiceAccount` |
| `aws:eks:pod-cpu-stress` | CPU stress on pods | `duration`, `percent`, `kubernetesServiceAccount` |
| `aws:eks:pod-memory-stress` | Memory stress on pods | `duration`, `percent`, `kubernetesServiceAccount` |
| `aws:eks:pod-io-stress` | I/O stress on pods | `duration`, `percent`, `kubernetesServiceAccount` |
| `aws:eks:pod-network-latency` | Inject latency | `duration`, `delayMilliseconds`, `jitterMilliseconds`, `sources` |
| `aws:eks:pod-network-packet-loss` | Inject packet loss | `duration`, `lossPercent`, `sources` |
| `aws:eks:pod-network-blackhole-port` | Blackhole port | `duration`, `port`, `protocol`, `trafficType` |
| `aws:eks:terminate-nodegroup-instances` | Terminate nodegroup instances | `instanceTerminationPercentage` |
| `aws:eks:inject-kubernetes-custom-resource` | Run ChaosMesh/Litmus | `kubernetesApiVersion`, `kubernetesKind`, `kubernetesNamespace`, `kubernetesSpec` |

### Storage Actions

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:ebs:pause-volume-io` | Pause EBS volume I/O | `duration` |
| `aws:ebs:volume-io-latency` | Inject EBS I/O latency | `duration`, `delayMilliseconds` |

### Serverless Actions

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:lambda:invocation-add-delay` | Delay Lambda execution | `duration`, `delayMilliseconds` |
| `aws:lambda:invocation-error` | Fail Lambda invocations | `duration` |
| `aws:lambda:invocation-http-integration-response` | Modify HTTP response | `duration`, `statusCode`, `body` |

### API Fault Injection Actions

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:fis:inject-api-internal-error` | Inject 500 errors on AWS APIs | `duration`, `service`, `operations`, `percentage` |
| `aws:fis:inject-api-throttle-error` | Inject throttling on AWS APIs | `duration`, `service`, `operations`, `percentage` |
| `aws:fis:inject-api-unavailable-error` | Inject 503 errors on AWS APIs | `duration`, `service`, `operations`, `percentage` |

### Replication Actions

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:dynamodb:global-table-pause-replication` | Pause DynamoDB Global Table replication | `duration` |
| `aws:s3:bucket-pause-replication` | Pause S3 bucket replication | `duration` |
| `aws:memorydb:multi-region-cluster-pause-replication` | Pause MemoryDB replication | `duration` |

### Other Actions

| Action | Description | Key Parameters |
|--------|-------------|----------------|
| `aws:fis:wait` | Pause between actions | `duration` |
| `aws:ssm:send-command` | Run SSM command on EC2 | `duration`, `documentArn`, `documentParameters` |
| `aws:ssm:start-automation-execution` | Run SSM automation | `duration`, `documentArn`, `documentParameters`, `maxDuration` |
| `aws:cloudwatch:assert-alarm-state` | Assert alarm is in expected state | `alarmArns`, `desiredState` |
| `aws:arc:start-zonal-autoshift` | Trigger ARC zonal autoshift | `duration` |
| `aws:elasticache:replicationgroup-interrupt-az-power` | Interrupt ElastiCache AZ power | `duration` |
| `aws:dsql:cluster-connection-failure` | Inject DSQL connection failure | `duration` |
| `aws:directconnect:virtual-interface-disconnect` | Disconnect Direct Connect VIF | `duration` |
| `aws:kinesis:stream-provisioned-throughput-exception` | Inject Kinesis throughput error | `duration`, `percentage` |
| `aws:kinesis:stream-expired-iterator-exception` | Inject Kinesis iterator error | `duration`, `percentage` |

---

## 10.2 CLI Quick Reference

### Template Management

```bash
# List all templates
aws fis list-experiment-templates --output table

# Create template from JSON
aws fis create-experiment-template --cli-input-json file://template.json

# Get template details
aws fis get-experiment-template --id EXT1234567890

# Update template
aws fis update-experiment-template --id EXT1234567890 \
  --description "Updated description"

# Delete template
aws fis delete-experiment-template --id EXT1234567890
```

### Experiment Execution

```bash
# Start experiment
aws fis start-experiment --experiment-template-id EXT1234567890

# Get experiment status
aws fis get-experiment --id EXP1234567890

# List running experiments
aws fis list-experiments \
  --query 'experiments[?state.status==`running`]' --output table

# List all experiments (recent first)
aws fis list-experiments \
  --query 'sort_by(experiments, &creationTime)[-5:]' --output table

# Stop experiment
aws fis stop-experiment --id EXP1234567890
```

### Discovery

```bash
# List all available actions
aws fis list-actions --query 'actions[].id' --output table

# Get action details
aws fis get-action --id aws:ec2:stop-instances

# List target resource types
aws fis list-target-resource-types --query 'targetResourceTypes[].resourceType' --output table

# Get target type details
aws fis get-target-resource-type --resource-type aws:ec2:instance
```

### Useful One-Liners

```bash
# Find all experiment templates tagged with a specific environment
aws fis list-experiment-templates \
  --query 'experimentTemplates[?tags.Environment==`production`]' --output table

# Get the last 5 experiments and their outcomes
aws fis list-experiments \
  --query 'sort_by(experiments, &creationTime)[-5:].{Id:id,Template:experimentTemplateId,Status:state.status,Start:startTime}' \
  --output table

# Export a template to JSON (useful for version control)
aws fis get-experiment-template --id EXT1234567890 \
  --query 'experimentTemplate' --output json > exported-template.json

# Count action-minutes for cost estimation
aws fis get-experiment --id EXP1234567890 \
  --query 'experiment.actions.*.{Action:actionId,State:state.status}' --output table
```

---

## 10.3 ISO 8601 Duration Quick Reference

| Duration | ISO 8601 |
|----------|----------|
| 30 seconds | `PT30S` |
| 1 minute | `PT1M` |
| 5 minutes | `PT5M` |
| 15 minutes | `PT15M` |
| 30 minutes | `PT30M` |
| 1 hour | `PT1H` |
| 2 hours | `PT2H` |
| 12 hours | `PT12H` (max) |

---

## 10.4 Official Documentation Links

| Resource | URL |
|----------|-----|
| FIS User Guide | https://docs.aws.amazon.com/fis/latest/userguide/what-is.html |
| Actions Reference | https://docs.aws.amazon.com/fis/latest/userguide/fis-actions-reference.html |
| Targets Reference | https://docs.aws.amazon.com/fis/latest/userguide/targets.html |
| IAM Roles for FIS | https://docs.aws.amazon.com/fis/latest/userguide/getting-started-iam-service-role.html |
| Experiment Templates | https://docs.aws.amazon.com/fis/latest/userguide/experiment-templates.html |
| Scenario Library | https://docs.aws.amazon.com/fis/latest/userguide/scenario-library.html |
| Pricing | https://aws.amazon.com/fis/pricing/ |
| Features Overview | https://aws.amazon.com/fis/features/ |
| FAQ | https://aws.amazon.com/fis/faqs/ |
| CloudFormation Reference | https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-fis-experimenttemplate.html |
| Terraform Resource | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/fis_experiment_template |
| Well-Architected: Chaos Engineering | https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_testing_resiliency_failure_injection_resiliency.html |

### Blogs and Tutorials

| Resource | URL |
|----------|-----|
| Chaos Engineering in the Cloud | https://aws.amazon.com/blogs/architecture/chaos-engineering-in-the-cloud/ |
| EBS Chaos Engineering with FIS | https://aws.amazon.com/blogs/storage/conducting-chaos-engineering-experiments-on-amazon-ebs-using-aws-fault-injection-simulator/ |
| FIS Scenarios Library Intro | https://aws.amazon.com/blogs/mt/bootstrap-your-chaos-engineering-journey-with-aws-fault-injection-service-scenarios-library/ |
| FIS Lambda Actions | https://aws.amazon.com/blogs/mt/introducing-aws-fault-injection-service-actions-to-inject-chaos-in-lambda-functions/ |
| Multi-Region FIS | https://aws.amazon.com/blogs/aws/use-aws-fault-injection-service-to-demonstrate-multi-region-and-multi-az-application-resilience/ |
| FIS with Bedrock (NLP) | https://aws.amazon.com/blogs/publicsector/chaos-engineering-made-clear-generate-aws-fis-experiments-using-natural-language-through-amazon-bedrock/ |

### Community and Training

| Resource | URL |
|----------|-----|
| AWS re:Post (FIS tag) | https://repost.aws/tags/TAbl-DsTlyTwCA7bfSe4CXWQ/aws-fault-injection-service |
| KodeKloud Chaos Engineering Course | https://kodekloud.com/courses/chaos-engineering |
| Pluralsight FIS Course | https://www.pluralsight.com/courses/hands-on-chaos-engineering-with-aws-fault-injection-simulator |
| Chaos Engineering Book (O'Reilly) | Search for "Chaos Engineering" by Casey Rosenthal & Nora Jones |
