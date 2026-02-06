# Module 11: Appendix

---

## A. Frequently Asked Questions

### Cost & Billing

**Q: How much will this runbook cost me in AWS charges?**
A: FIS costs are minimal (~$5-15 total for all labs). The main cost driver is the resources you run: EC2 `t3.micro` instances (~$0.01/hr each), Aurora `db.t3.medium` (~$0.07/hr each), and EKS cluster (~$0.10/hr for the control plane + node instances). If you tear down resources after each lab session, budget $20-50 total.

**Q: Is there an AWS Free Tier for FIS?**
A: AWS Free Tier includes some FIS usage for eligible accounts. Check current Free Tier terms at https://aws.amazon.com/free/.

**Q: Will FIS experiments affect my AWS bill beyond the FIS charges?**
A: Yes. If an experiment causes auto-scaling (e.g., ASG launches replacement instances), you pay for those additional resources. If it triggers data transfer (e.g., cross-AZ failover), standard data transfer rates apply.

### Safety

**Q: Can FIS accidentally destroy my production data?**
A: FIS respects the IAM permission boundary you configure. If you don't grant `ec2:TerminateInstances` to the experiment role, FIS cannot terminate instances. Scope permissions tightly. For databases, `aws:rds:failover-db-cluster` promotes a reader -- it doesn't delete data.

**Q: What if an experiment gets stuck and won't stop?**
A: Use `aws fis stop-experiment --id EXPERIMENT_ID`. If that fails, the experiment will auto-terminate when its maximum duration is reached (12 hours max). For network disruption experiments, verify NACLs are restored and manually fix if needed.

**Q: Can I use FIS in production?**
A: Yes, that's its intended use. AWS itself uses FIS-like internal tools for production resilience testing. Start with `COUNT(1)` targets, always use stop conditions, and follow the safety pyramid in Module 9.

**Q: What happens if I lose internet connectivity during an experiment?**
A: The experiment continues running in AWS. Stop conditions still work. When you regain connectivity, check experiment status and stop it if needed. This is why stop conditions are critical -- they're your automated safety net.

### Technical

**Q: Can I run multiple experiments simultaneously?**
A: Yes, FIS supports concurrent experiments. However, avoid targeting the same resources from multiple experiments as the combined effects are unpredictable.

**Q: What's the maximum experiment duration?**
A: Individual actions can run up to 12 hours. An experiment's total duration depends on action sequencing.

**Q: Does FIS work with non-AWS resources?**
A: Not directly. For non-AWS resources (on-premises, other clouds), use `aws:ssm:send-command` to run custom scripts on SSM-managed instances, or trigger external tools via Lambda.

**Q: How does FIS differ from Chaos Monkey / Litmus / ChaosMesh?**
A: FIS is AWS-native, agentless (for most actions), and integrated with IAM for permission control. Open-source tools are cloud-agnostic but require agent deployment and management. You can use both -- FIS for AWS-level experiments, ChaosMesh/Litmus for Kubernetes-level experiments. FIS even supports running ChaosMesh/Litmus via `aws:eks:inject-kubernetes-custom-resource`.

**Q: Can I undo a `terminate-instances` action?**
A: No. Termination is irreversible. If you need reversibility, use `stop-instances` with `startInstancesAfterDuration` instead. Only use terminate for testing Auto Scaling Group replacement behavior.

---

## B. Capstone Project: Multi-AZ Resilience Game Day

**Time: 4 hours**

This project combines everything you've learned into a realistic Game Day simulation.

### Scenario

You operate a web application with:
- 2 EC2 web servers across 2 AZs (from the lab stack)
- An Aurora database cluster (from Module 5)
- CloudWatch monitoring and alarms

**Mission**: Simulate a partial AZ-a outage and verify the system stays available.

### Step 1: Build the Monitoring Layer

Create CloudWatch alarms as stop conditions:

```bash
# Alarm 1: EC2 health check
aws cloudwatch put-metric-alarm \
  --alarm-name capstone-ec2-health \
  --metric-name StatusCheckFailed \
  --namespace AWS/EC2 \
  --statistic Maximum \
  --period 60 \
  --threshold 2 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --treat-missing-data notBreaching

# Alarm 2: Manual safety override
aws cloudwatch put-metric-alarm \
  --alarm-name capstone-manual-stop \
  --metric-name CapstoneErrorRate \
  --namespace Capstone \
  --statistic Average \
  --period 60 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --treat-missing-data notBreaching
```

### Step 2: Create the Multi-Phase Experiment

Save as `capstone-template.json`:
```json
{
  "description": "Capstone: Multi-AZ resilience Game Day - partial AZ-a outage",
  "targets": {
    "az-a-instances": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": {
        "Environment": ["fis-lab"]
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
    "subnet-a": {
      "resourceType": "aws:ec2:subnet",
      "resourceTags": {
        "Name": ["fis-lab-public-a"]
      },
      "selectionMode": "ALL"
    },
    "aurora-cluster": {
      "resourceType": "aws:rds:cluster",
      "resourceArns": [
        "arn:aws:rds:us-east-1:YOUR_ACCOUNT_ID:cluster:fis-lab-aurora"
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "Phase1_DisruptNetworkAZa": {
      "actionId": "aws:network:disrupt-connectivity",
      "description": "Disrupt network in AZ-a subnet",
      "parameters": {
        "duration": "PT5M",
        "scope": "all"
      },
      "targets": {
        "Subnets": "subnet-a"
      }
    },
    "Phase1_StopEC2AZa": {
      "actionId": "aws:ec2:stop-instances",
      "description": "Stop EC2 instances in AZ-a",
      "parameters": {
        "startInstancesAfterDuration": "PT7M"
      },
      "targets": {
        "Instances": "az-a-instances"
      }
    },
    "Phase2_FailoverDB": {
      "actionId": "aws:rds:failover-db-cluster",
      "description": "Failover Aurora after network disruption starts",
      "startAfter": ["Phase1_DisruptNetworkAZa"],
      "targets": {
        "Clusters": "aurora-cluster"
      }
    },
    "Phase3_ObserveRecovery": {
      "actionId": "aws:fis:wait",
      "description": "Observe system recovery",
      "parameters": {
        "duration": "PT5M"
      },
      "startAfter": ["Phase1_DisruptNetworkAZa", "Phase1_StopEC2AZa", "Phase2_FailoverDB"]
    }
  },
  "stopConditions": [
    {
      "source": "aws:cloudwatch:alarm",
      "value": "arn:aws:cloudwatch:us-east-1:YOUR_ACCOUNT_ID:alarm:capstone-manual-stop"
    }
  ],
  "roleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/FISExperimentRole",
  "tags": {
    "Project": "capstone",
    "Environment": "fis-lab"
  }
}
```

**Execution timeline:**

```
T+0        T+1min      T+5min      T+7min       T+12min
 |           |           |           |             |
 ├── Phase 1: Disrupt Network AZ-a (5 min) ──┤
 ├── Phase 1: Stop EC2 AZ-a ──────────────────────── restart ──┤
 |           ├── Phase 2: DB Failover (~30s) ──┤
 |           |           |           ├── Phase 3: Observe (5 min) ──┤
 |           |           |           |             |
 ▼           ▼           ▼           ▼             ▼
 START    DB failover  Network     EC2 restart    COMPLETE
          triggers     restores
```

### Step 3: Run and Observe

```bash
# Create template
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://capstone-template.json \
  --query 'experimentTemplate.id' --output text)

# Open monitoring in separate terminals:
# Terminal 1: Experiment status
watch -n 5 "aws fis get-experiment --id \$EXPERIMENT_ID \
  --query 'experiment.actions' --output json | jq ."

# Terminal 2: EC2 instance states
watch -n 5 "aws ec2 describe-instances \
  --filters 'Name=tag:Environment,Values=fis-lab' \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==\`Name\`].Value|[0],State:State.Name,AZ:Placement.AvailabilityZone}' \
  --output table"

# Terminal 3: Aurora cluster status
watch -n 10 "aws rds describe-db-clusters \
  --db-cluster-identifier fis-lab-aurora \
  --query 'DBClusters[0].{Status:Status,Members:DBClusterMembers[].{Id:DBInstanceIdentifier,Writer:IsClusterWriter}}' \
  --output json"

# Start the experiment
EXPERIMENT_ID=$(aws fis start-experiment \
  --experiment-template-id "$TEMPLATE_ID" \
  --query 'experiment.id' --output text)

echo "CAPSTONE EXPERIMENT STARTED: $EXPERIMENT_ID"
echo "$(date): Experiment running. Monitor the terminals."
```

### Step 4: Document Your Findings

Fill out this report:

```markdown
# Capstone Game Day Report

**Date**: ____
**Experiment ID**: ____
**Duration**: ____

## Results

| Metric | Expected | Actual |
|--------|----------|--------|
| Network disruption detected in | < 60s | ____ |
| EC2 stop detected in | < 60s | ____ |
| DB failover completed in | < 30s | ____ |
| Network restored after | 5 min | ____ |
| EC2 restarted after | 7 min | ____ |
| Total user-visible impact | < 2 min | ____ |

## Observations
- What broke: ____
- What held: ____
- Surprises: ____

## Action Items
1. ____
2. ____
3. ____
```

### Step 5: Cleanup

```bash
# Delete experiment template
aws fis delete-experiment-template --id "$TEMPLATE_ID"

# Delete CloudWatch alarms
aws cloudwatch delete-alarms \
  --alarm-names capstone-ec2-health capstone-manual-stop

# Delete RDS stack
aws cloudformation delete-stack --stack-name fis-lab-rds

# Delete core lab stack
aws cloudformation delete-stack --stack-name fis-lab-environment

# Wait for deletion
aws cloudformation wait stack-delete-complete --stack-name fis-lab-rds
aws cloudformation wait stack-delete-complete --stack-name fis-lab-environment

# Delete FIS experiment role
aws iam delete-role-policy \
  --role-name FISExperimentRole \
  --policy-name FISExperimentPolicy
aws iam delete-role --role-name FISExperimentRole

# Delete SSM document (if created in Module 8)
aws ssm delete-document --name FIS-CPUStress 2>/dev/null

# Final verification -- should return empty results
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=fis-lab" \
  --query 'Reservations[].Instances[].InstanceId' --output text

echo "Cleanup complete."
```

---

## C. Sample Experiment Template Repository

Use these as starting points for your own experiments. Replace `YOUR_ACCOUNT_ID` and resource identifiers.

### Template: Canary Instance Kill

```json
{
  "description": "Kill a single instance to test ASG replacement",
  "targets": {
    "instance": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": { "ChaosReady": ["true"] },
      "filters": [{ "path": "State.Name", "values": ["running"] }],
      "selectionMode": "COUNT(1)"
    }
  },
  "actions": {
    "TerminateOne": {
      "actionId": "aws:ec2:terminate-instances",
      "targets": { "Instances": "instance" }
    }
  },
  "stopConditions": [
    { "source": "aws:cloudwatch:alarm", "value": "arn:aws:cloudwatch:REGION:ACCOUNT:alarm:ALARM_NAME" }
  ],
  "roleArn": "arn:aws:iam::ACCOUNT:role/FISExperimentRole"
}
```

### Template: Spot Instance Interruption Test

```json
{
  "description": "Send Spot interruption notice to test graceful handling",
  "targets": {
    "spots": {
      "resourceType": "aws:ec2:spot-instance",
      "resourceTags": { "ChaosReady": ["true"] },
      "selectionMode": "COUNT(1)"
    }
  },
  "actions": {
    "InterruptSpot": {
      "actionId": "aws:ec2:send-spot-instance-interruptions",
      "parameters": { "durationBeforeInterruption": "PT2M" },
      "targets": { "SpotInstances": "spots" }
    }
  },
  "stopConditions": [{ "source": "none" }],
  "roleArn": "arn:aws:iam::ACCOUNT:role/FISExperimentRole"
}
```

### Template: Lambda Latency Injection

```json
{
  "description": "Add 500ms latency to Lambda function invocations",
  "targets": {
    "lambda-fn": {
      "resourceType": "aws:lambda:function",
      "resourceArns": ["arn:aws:lambda:REGION:ACCOUNT:function:FUNCTION_NAME"],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "AddLatency": {
      "actionId": "aws:lambda:invocation-add-delay",
      "parameters": {
        "duration": "PT5M",
        "delayMilliseconds": "500"
      },
      "targets": { "Functions": "lambda-fn" }
    }
  },
  "stopConditions": [{ "source": "none" }],
  "roleArn": "arn:aws:iam::ACCOUNT:role/FISExperimentRole"
}
```

### Template: DynamoDB Global Table Replication Pause

```json
{
  "description": "Pause DynamoDB Global Table replication to test eventual consistency",
  "targets": {
    "global-table": {
      "resourceType": "aws:dynamodb:global-table",
      "resourceArns": ["arn:aws:dynamodb:REGION:ACCOUNT:table/TABLE_NAME"],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "PauseReplication": {
      "actionId": "aws:dynamodb:global-table-pause-replication",
      "parameters": { "duration": "PT5M" },
      "targets": { "GlobalTables": "global-table" }
    }
  },
  "stopConditions": [{ "source": "none" }],
  "roleArn": "arn:aws:iam::ACCOUNT:role/FISExperimentRole"
}
```
