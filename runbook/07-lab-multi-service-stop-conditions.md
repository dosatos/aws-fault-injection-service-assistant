# Module 7: Lab 5 - Multi-Service Experiments with Stop Conditions

**Time: 3 hours**

**Learning Objectives:**
- Design experiments with multiple actions (parallel and sequential)
- Implement stop conditions using CloudWatch alarms
- Build a realistic multi-layer failure scenario
- Understand action sequencing with `startAfter`

**Prerequisites:** Modules 3-5 completed

---

## Objective

Build a complex experiment that simulates a partial AZ outage: stop an EC2 instance AND disrupt subnet connectivity simultaneously, with CloudWatch alarm guardrails that abort the experiment if error rates exceed thresholds.

**Hypothesis:** "When Subnet A experiences network disruption and one EC2 instance stops, our system's error rate stays below 5% because traffic shifts to Subnet B."

---

## 7.1 Create the Stop Condition (CloudWatch Alarm)

Stop conditions are CloudWatch alarms that act as circuit breakers. If the alarm enters `ALARM` state, FIS stops the experiment immediately.

```bash
# Create a simple alarm based on EC2 status checks
# In production, you'd use application-level metrics (5xx rate, latency, etc.)
aws cloudwatch put-metric-alarm \
  --alarm-name fis-lab-safety-alarm \
  --alarm-description "FIS safety stop - triggers if too many instances are unhealthy" \
  --metric-name StatusCheckFailed \
  --namespace AWS/EC2 \
  --statistic Maximum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=AutoScalingGroupName,Value=fis-lab-asg \
  --treat-missing-data notBreaching \
  --actions-enabled

# For a more realistic setup, create an alarm on application error rate:
aws cloudwatch put-metric-alarm \
  --alarm-name fis-lab-error-rate-alarm \
  --alarm-description "FIS safety stop - error rate too high" \
  --metric-name 5XXError \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --period 60 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --treat-missing-data notBreaching \
  --actions-enabled
```

> **Pro Tip**: For this lab, if you don't have an ALB, create a simple synthetic alarm that you can manually trigger:

```bash
# Create alarm on a custom metric you can control
aws cloudwatch put-metric-alarm \
  --alarm-name fis-lab-manual-stop \
  --metric-name FISLabErrorRate \
  --namespace FISLab \
  --statistic Average \
  --period 60 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --treat-missing-data notBreaching

# To trigger this alarm manually during an experiment:
aws cloudwatch set-alarm-state \
  --alarm-name fis-lab-manual-stop \
  --state-value ALARM \
  --state-reason "Manual trigger to test FIS stop condition"
```

---

## 7.2 Multi-Action Experiment with Sequencing

FIS supports two execution patterns:

```
Parallel (default):              Sequential (using startAfter):
┌──────────────┐                 ┌──────────────┐
│  Action A    │                 │  Action A    │
│  (5 min)     │                 │  (5 min)     │
├──────────────┤                 └──────┬───────┘
│  Action B    │                        │ startAfter: A
│  (5 min)     │                 ┌──────▼───────┐
└──────────────┘                 │  Action B    │
Both start at T+0               │  (5 min)     │
                                 └──────────────┘
                                 B starts after A completes
```

### The Experiment Template

Save as `lab7-template.json`:
```json
{
  "description": "Lab 7: Multi-service failure with stop conditions - partial AZ outage simulation",
  "targets": {
    "ec2-target": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": {
        "Environment": ["fis-lab"],
        "Role": ["webserver"]
      },
      "filters": [
        {
          "path": "State.Name",
          "values": ["running"]
        },
        {
          "path": "Placement.AvailabilityZone",
          "values": ["us-east-1a"]
        }
      ],
      "selectionMode": "ALL"
    },
    "subnet-target": {
      "resourceType": "aws:ec2:subnet",
      "resourceTags": {
        "Name": ["fis-lab-public-a"]
      },
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "DisruptNetwork": {
      "actionId": "aws:network:disrupt-connectivity",
      "description": "Phase 1: Disrupt Subnet A connectivity",
      "parameters": {
        "duration": "PT5M",
        "scope": "all"
      },
      "targets": {
        "Subnets": "subnet-target"
      }
    },
    "StopInstance": {
      "actionId": "aws:ec2:stop-instances",
      "description": "Phase 1: Stop EC2 in AZ-a (runs in parallel with DisruptNetwork)",
      "parameters": {
        "startInstancesAfterDuration": "PT5M"
      },
      "targets": {
        "Instances": "ec2-target"
      }
    },
    "WaitForRecovery": {
      "actionId": "aws:fis:wait",
      "description": "Phase 2: Wait and observe system recovery",
      "parameters": {
        "duration": "PT3M"
      },
      "startAfter": ["DisruptNetwork", "StopInstance"]
    }
  },
  "stopConditions": [
    {
      "source": "aws:cloudwatch:alarm",
      "value": "arn:aws:cloudwatch:us-east-1:YOUR_ACCOUNT_ID:alarm:fis-lab-manual-stop"
    }
  ],
  "roleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/FISExperimentRole",
  "tags": {
    "Lab": "7",
    "Environment": "fis-lab"
  }
}
```

**What this does:**
1. **Phase 1** (parallel): Disrupts Subnet A network AND stops EC2 instances in AZ-a simultaneously
2. **Phase 2** (sequential): After both Phase 1 actions complete, waits 3 minutes to observe recovery
3. **Safety**: If the CloudWatch alarm triggers at any point, everything stops immediately

### Create and Run

```bash
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://lab7-template.json \
  --query 'experimentTemplate.id' --output text)

echo "Template: $TEMPLATE_ID"

EXPERIMENT_ID=$(aws fis start-experiment \
  --experiment-template-id "$TEMPLATE_ID" \
  --query 'experiment.id' --output text)

echo "Experiment: $EXPERIMENT_ID"
```

---

## 7.3 Monitoring the Multi-Action Experiment

```bash
# Watch experiment progress (actions and their states)
watch -n 5 "aws fis get-experiment --id $EXPERIMENT_ID \
  --query 'experiment.{State:state,Actions:actions}' --output json | jq ."

# Watch EC2 states
watch -n 5 "aws ec2 describe-instances \
  --filters 'Name=tag:Environment,Values=fis-lab' \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==\`Name\`].Value|[0],State:State.Name,AZ:Placement.AvailabilityZone}' \
  --output table"
```

### Test the Stop Condition

While the experiment is running, trigger the alarm manually:

```bash
# This should immediately stop the experiment
aws cloudwatch set-alarm-state \
  --alarm-name fis-lab-manual-stop \
  --state-value ALARM \
  --state-reason "Testing FIS stop condition"

# Verify experiment was stopped
aws fis get-experiment --id "$EXPERIMENT_ID" \
  --query 'experiment.state' --output json
```

**Expected output:**
```json
{
  "status": "stopped",
  "reason": "Stop condition triggered"
}
```

---

## 7.4 Terraform

```hcl
resource "aws_cloudwatch_metric_alarm" "fis_safety" {
  alarm_name          = "fis-lab-safety-alarm"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FISLabErrorRate"
  namespace           = "FISLab"
  period              = 60
  statistic           = "Average"
  threshold           = 50
  alarm_description   = "Safety stop for FIS experiments"
  treat_missing_data  = "notBreaching"
}

resource "aws_fis_experiment_template" "lab7_multi_service" {
  description = "Lab 7: Multi-service failure with stop conditions"
  role_arn    = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/FISExperimentRole"

  stop_condition {
    source = "aws:cloudwatch:alarm"
    value  = aws_cloudwatch_metric_alarm.fis_safety.arn
  }

  # Phase 1: Parallel actions
  action {
    name        = "DisruptNetwork"
    action_id   = "aws:network:disrupt-connectivity"
    description = "Disrupt Subnet A connectivity"

    parameter {
      key   = "duration"
      value = "PT5M"
    }

    parameter {
      key   = "scope"
      value = "all"
    }

    target {
      key   = "Subnets"
      value = "subnet-target"
    }
  }

  action {
    name        = "StopInstance"
    action_id   = "aws:ec2:stop-instances"
    description = "Stop EC2 in AZ-a"

    parameter {
      key   = "startInstancesAfterDuration"
      value = "PT5M"
    }

    target {
      key   = "Instances"
      value = "ec2-target"
    }
  }

  # Phase 2: Sequential wait
  action {
    name        = "WaitForRecovery"
    action_id   = "aws:fis:wait"
    description = "Observe recovery"
    start_after = ["DisruptNetwork", "StopInstance"]

    parameter {
      key   = "duration"
      value = "PT3M"
    }
  }

  target {
    name           = "ec2-target"
    resource_type  = "aws:ec2:instance"
    selection_mode = "ALL"

    resource_tag {
      key    = "Environment"
      value  = "fis-lab"
    }

    filter {
      path   = "State.Name"
      values = ["running"]
    }

    filter {
      path   = "Placement.AvailabilityZone"
      values = ["us-east-1a"]
    }
  }

  target {
    name           = "subnet-target"
    resource_type  = "aws:ec2:subnet"
    selection_mode = "ALL"

    resource_tag {
      key    = "Name"
      value  = "fis-lab-public-a"
    }
  }

  tags = {
    Lab         = "7"
    Environment = "fis-lab"
  }
}
```

---

## 7.5 Stop Condition Best Practices

| Guideline | Details |
|-----------|---------|
| **Always use stop conditions in production** | `"source": "none"` is only for isolated lab environments |
| **Use application-level metrics** | 5xx rate, latency p99, error count -- not just infra metrics |
| **Set thresholds below SLA** | If SLA is 99.9%, alarm at 99.5% to give margin |
| **Use multiple stop conditions** | One per critical metric (error rate AND latency AND queue depth) |
| **Test the stop condition first** | Manually trigger it before running experiments to confirm it works |
| **Short evaluation periods** | Use 1 evaluation period with 60s period for fast response |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Stop condition doesn't trigger | Alarm not in correct account/region | Verify alarm ARN matches experiment region |
| Stop condition ignored | Alarm in `INSUFFICIENT_DATA` state | Set `treat_missing_data: notBreaching` |
| `startAfter` action never starts | Previous action failed | Check dependent action state -- if it fails, sequenced actions are skipped |
| Experiment takes too long | Wait action + action durations stacking | Sum up: Phase 1 (max parallel duration) + Phase 2 (wait) = total time |

---

## Cleanup

```bash
aws fis delete-experiment-template --id "$TEMPLATE_ID"
aws cloudwatch delete-alarms --alarm-names fis-lab-manual-stop fis-lab-safety-alarm

# Reset alarm state if needed
aws cloudwatch set-alarm-state \
  --alarm-name fis-lab-manual-stop \
  --state-value OK \
  --state-reason "Reset after lab"
```

---

## What You Learned

- How to combine multiple actions in parallel and sequential phases
- How `startAfter` creates execution dependencies between actions
- How stop conditions work as automated safety circuit breakers
- How to use `aws:fis:wait` for observation periods
- Best practices for production stop conditions

---

## Knowledge Checkpoint

1. If Action A (5 min) and Action B (3 min) run in parallel, how long is Phase 1?
2. What happens to Action C (startAfter: [A, B]) if Action A fails?
3. How quickly can a stop condition react to an alarm state change?
4. Name 3 metrics you'd use as stop conditions for a web application.

<details>
<summary>Answers</summary>

1. 5 minutes (the duration of the longest parallel action).
2. Action C may be skipped or the experiment may fail. FIS won't start sequenced actions if their dependencies failed.
3. FIS checks stop conditions roughly every 30 seconds. Combined with CloudWatch's evaluation period (60s default), worst case is ~90 seconds.
4. HTTP 5xx error rate, p99 latency, and request queue depth (or active connection count, error budget burn rate, etc.).

</details>
