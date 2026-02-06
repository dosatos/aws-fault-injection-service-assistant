# Module 8: Advanced Topics

**Time: 4 hours**

**Learning Objectives:**
- Create custom FIS actions using SSM documents
- Implement tag-based targeting strategies for complex environments
- Integrate FIS with CloudWatch dashboards and observability tools
- Automate experiment execution with EventBridge and Lambda
- Define experiments as Infrastructure as Code (CloudFormation and Terraform)

---

## 8.1 Custom Actions with SSM Documents

FIS supports two SSM-based actions for extending beyond built-in capabilities:

| Action | Use Case |
|--------|----------|
| `aws:ssm:send-command` | Run shell commands on EC2 instances (via SSM Run Command) |
| `aws:ssm:start-automation-execution` | Run multi-step SSM Automation documents |

### Example: Custom CPU Stress on EC2 via SSM

First, ensure your EC2 instances have the SSM agent installed and an instance profile with `AmazonSSMManagedInstanceCore` policy.

**SSM Document** (`fis-cpu-stress.yaml`):
```yaml
schemaVersion: '2.2'
description: 'FIS Lab: Custom CPU stress test'
parameters:
  DurationSeconds:
    type: String
    default: '120'
    description: Duration of stress test in seconds
  CPUPercent:
    type: String
    default: '80'
    description: Target CPU utilization percentage
mainSteps:
  - action: aws:runShellScript
    name: runCPUStress
    inputs:
      runCommand:
        - |
          # Install stress-ng if not present
          which stress-ng || sudo yum install -y stress-ng || sudo apt-get install -y stress-ng

          # Get number of CPUs
          NCPU=$(nproc)

          # Run stress test
          stress-ng --cpu $NCPU --cpu-load {{ CPUPercent }} --timeout {{ DurationSeconds }}s

          echo "CPU stress test completed"
```

```bash
# Create the SSM document
aws ssm create-document \
  --name "FIS-CPUStress" \
  --document-type "Command" \
  --content file://fis-cpu-stress.yaml \
  --document-format YAML
```

**FIS template using the custom SSM action** (`lab8-ssm-template.json`):
```json
{
  "description": "Lab 8: Custom CPU stress via SSM",
  "targets": {
    "ec2-target": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": {
        "Environment": ["fis-lab"]
      },
      "selectionMode": "COUNT(1)"
    }
  },
  "actions": {
    "CustomCPUStress": {
      "actionId": "aws:ssm:send-command",
      "parameters": {
        "duration": "PT5M",
        "documentArn": "arn:aws:ssm:us-east-1:YOUR_ACCOUNT_ID:document/FIS-CPUStress",
        "documentParameters": "{\"DurationSeconds\":\"120\",\"CPUPercent\":\"80\"}"
      },
      "targets": {
        "Instances": "ec2-target"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/FISExperimentRole",
  "tags": {
    "Lab": "8",
    "Environment": "fis-lab"
  }
}
```

> **Pro Tip**: SSM Run Command has its own execution timeout. Set the FIS action `duration` longer than the SSM document's execution time to avoid premature termination.

### Other Custom Action Ideas

| SSM Document | What It Tests |
|-------------|---------------|
| Fill disk to 90% | Application behavior when storage is full |
| Kill a specific process | Critical daemon failure (e.g., nginx, java) |
| Corrupt DNS resolution | Dependency resolution failures |
| Drop iptables traffic on specific port | Simulate service dependency outage |
| Modify `/etc/hosts` | DNS override for testing failover |

---

## 8.2 Complex Targeting Strategies

### Tag-Based Selection Patterns

```
Strategy 1: Environment-based
┌─────────────────────────────────┐
│ Tag: Environment = staging      │ ← Target all staging resources
│ Selection: ALL                  │
└─────────────────────────────────┘

Strategy 2: Percentage of a tier
┌─────────────────────────────────┐
│ Tag: Tier = frontend            │
│ Tag: Environment = production   │
│ Selection: PERCENT(25)          │ ← 25% of prod frontend
└─────────────────────────────────┘

Strategy 3: AZ-specific
┌─────────────────────────────────┐
│ Tag: Environment = production   │
│ Filter: AZ = us-east-1a        │ ← Only resources in AZ-a
│ Selection: ALL                  │
└─────────────────────────────────┘
```

**Multi-tag targeting** (all tags must match -- AND logic):
```json
{
  "resourceTags": {
    "Environment": ["production"],
    "Tier": ["frontend"],
    "Team": ["platform"]
  },
  "filters": [
    {
      "path": "State.Name",
      "values": ["running"]
    }
  ],
  "selectionMode": "PERCENT(10)"
}
```

### Tagging Strategy for FIS

| Tag | Purpose | Example Values |
|-----|---------|---------------|
| `Environment` | Scope experiments to env | `dev`, `staging`, `production` |
| `Tier` | Target specific app layers | `frontend`, `backend`, `database`, `cache` |
| `ChaosReady` | Opt-in for chaos experiments | `true`, `false` |
| `Team` | Ownership for blast radius control | `platform`, `payments`, `search` |

> **Best Practice**: Add a `ChaosReady: true` tag to resources that have been validated for chaos experiments. Use this in your FIS targets to prevent accidental targeting of unprepared resources.

---

## 8.3 Observability Integration

### CloudWatch Dashboard for FIS

Create a dashboard that shows experiment impact at a glance:

```bash
aws cloudwatch put-dashboard \
  --dashboard-name FIS-Lab-Dashboard \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "title": "EC2 Instance Status",
          "metrics": [
            ["AWS/EC2", "StatusCheckFailed", "InstanceId", "i-YOUR_INSTANCE_ID"]
          ],
          "period": 60,
          "stat": "Maximum"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "Network In/Out",
          "metrics": [
            ["AWS/EC2", "NetworkIn", "InstanceId", "i-YOUR_INSTANCE_ID"],
            ["AWS/EC2", "NetworkOut", "InstanceId", "i-YOUR_INSTANCE_ID"]
          ],
          "period": 60,
          "stat": "Average"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "RDS Connections",
          "metrics": [
            ["AWS/RDS", "DatabaseConnections", "DBClusterIdentifier", "fis-lab-aurora"]
          ],
          "period": 60,
          "stat": "Average"
        }
      }
    ]
  }'
```

### CloudWatch Logs Integration

FIS experiment logs can be sent to CloudWatch Logs for analysis:

```json
{
  "logConfiguration": {
    "cloudWatchLogsConfiguration": {
      "logGroupArn": "arn:aws:logs:us-east-1:YOUR_ACCOUNT_ID:log-group:/fis/experiments:*"
    },
    "logSchemaVersion": 2
  }
}
```

### Third-Party APM Integration

| Tool | Integration Method |
|------|-------------------|
| **Datadog** | Use CloudWatch integration + FIS experiment events as markers/annotations |
| **Grafana** | CloudWatch data source + annotation API for experiment start/stop |
| **New Relic** | CloudWatch Metric Streams + deployment markers via API |
| **PagerDuty** | EventBridge -> SNS -> PagerDuty integration for experiment notifications |

**Datadog example** -- annotate dashboards with experiment events:
```bash
# Send experiment start event to Datadog
curl -X POST "https://api.datadoghq.com/api/v1/events" \
  -H "DD-API-KEY: $DD_API_KEY" \
  -d '{
    "title": "FIS Experiment Started",
    "text": "Experiment '"$EXPERIMENT_ID"' running template '"$TEMPLATE_ID"'",
    "tags": ["source:fis", "env:staging"],
    "alert_type": "warning"
  }'
```

---

## 8.4 Automating Experiments with EventBridge + Lambda

### Scheduled Experiments

Run experiments on a recurring schedule (e.g., every Tuesday at 2 PM):

```json
{
  "ScheduleExpression": "cron(0 14 ? * TUE *)",
  "Target": {
    "Arn": "arn:aws:lambda:us-east-1:YOUR_ACCOUNT_ID:function:StartFISExperiment",
    "Id": "WeeklyFISExperiment"
  }
}
```

**Lambda function** to start experiments:
```python
# start_fis_experiment.py
import boto3
import os
import json

def handler(event, context):
    fis = boto3.client('fis')
    template_id = os.environ['EXPERIMENT_TEMPLATE_ID']

    response = fis.start_experiment(
        experimentTemplateId=template_id,
        tags={
            'TriggeredBy': 'EventBridge-Schedule',
            'Timestamp': context.invoked_function_arn
        }
    )

    experiment_id = response['experiment']['id']
    print(f"Started experiment: {experiment_id}")

    return {
        'statusCode': 200,
        'body': json.dumps({
            'experimentId': experiment_id,
            'templateId': template_id
        })
    }
```

### Event-Driven Experiments

Trigger experiments based on events (e.g., after a deployment):

```json
{
  "source": ["aws.codedeploy"],
  "detail-type": ["CodeDeploy Deployment State-change Notification"],
  "detail": {
    "state": ["SUCCESS"],
    "application": ["my-application"]
  }
}
```

### CloudFormation for EventBridge + Lambda

```yaml
Resources:
  FISSchedulerFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: StartFISExperiment
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt FISSchedulerRole.Arn
      Environment:
        Variables:
          EXPERIMENT_TEMPLATE_ID: !Ref ExperimentTemplateId
      Code:
        ZipFile: |
          import boto3, os, json
          def handler(event, context):
              fis = boto3.client('fis')
              resp = fis.start_experiment(
                  experimentTemplateId=os.environ['EXPERIMENT_TEMPLATE_ID'])
              return {'experimentId': resp['experiment']['id']}

  ScheduleRule:
    Type: AWS::Events::Rule
    Properties:
      Name: weekly-fis-experiment
      ScheduleExpression: "cron(0 14 ? * TUE *)"
      State: ENABLED
      Targets:
        - Arn: !GetAtt FISSchedulerFunction.Arn
          Id: FISSchedulerTarget

  SchedulePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref FISSchedulerFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt ScheduleRule.Arn
```

---

## 8.5 Experiments as Infrastructure as Code

### CloudFormation: `AWS::FIS::ExperimentTemplate`

```yaml
Resources:
  ProductionResilienceTest:
    Type: AWS::FIS::ExperimentTemplate
    Properties:
      Description: "Production resilience test: AZ-a partial outage"
      RoleArn: !GetAtt FISExperimentRole.Arn
      Tags:
        Environment: production
        Team: platform
      StopConditions:
        - Source: aws:cloudwatch:alarm
          Value: !GetAtt ErrorRateAlarm.Arn
      Targets:
        ec2Target:
          ResourceType: aws:ec2:instance
          ResourceTags:
            Environment:
              - production
            Tier:
              - frontend
          Filters:
            - Path: Placement.AvailabilityZone
              Values:
                - us-east-1a
          SelectionMode: PERCENT(25)
      Actions:
        StopInstances:
          ActionId: aws:ec2:stop-instances
          Parameters:
            startInstancesAfterDuration: PT10M
          Targets:
            Instances: ec2Target
```

### Terraform: `aws_fis_experiment_template`

```hcl
resource "aws_fis_experiment_template" "production_resilience" {
  description = "Production resilience test: AZ-a partial outage"
  role_arn    = aws_iam_role.fis_experiment.arn

  stop_condition {
    source = "aws:cloudwatch:alarm"
    value  = aws_cloudwatch_metric_alarm.error_rate.arn
  }

  action {
    name      = "StopInstances"
    action_id = "aws:ec2:stop-instances"

    parameter {
      key   = "startInstancesAfterDuration"
      value = "PT10M"
    }

    target {
      key   = "Instances"
      value = "ec2-target"
    }
  }

  target {
    name           = "ec2-target"
    resource_type  = "aws:ec2:instance"
    selection_mode = "PERCENT(25)"

    resource_tag {
      key    = "Environment"
      value  = "production"
    }

    resource_tag {
      key    = "Tier"
      value  = "frontend"
    }

    filter {
      path   = "Placement.AvailabilityZone"
      values = ["us-east-1a"]
    }
  }

  tags = {
    Environment = "production"
    Team        = "platform"
    ManagedBy   = "terraform"
  }
}
```

> **Pro Tip**: Store experiment templates in the same repo as your infrastructure code. Review them in PRs just like any other infrastructure change.

---

## Knowledge Checkpoint

1. What's the difference between `aws:ssm:send-command` and `aws:ssm:start-automation-execution`?
2. How would you target only resources that have been validated for chaos testing?
3. Name two ways to trigger FIS experiments automatically.
4. Why should experiment templates be stored as IaC?

<details>
<summary>Answers</summary>

1. `send-command` runs shell commands on EC2 instances (Run Command). `start-automation-execution` runs multi-step SSM Automation workflows that can span multiple AWS services and include conditional logic.
2. Add a `ChaosReady: true` tag to validated resources and include it in the target's `resourceTags` filter.
3. EventBridge scheduled rules (cron) and event-driven rules (e.g., post-deployment triggers).
4. Version control, code review, reproducibility, and consistency across environments. Experiment templates should go through the same review process as infrastructure changes.

</details>
