# Module 3: Lab 1 - EC2 Stop/Start

**Time: 2 hours**

**Learning Objectives:**
- Create your first FIS experiment template
- Execute an experiment via Console, CLI, and Terraform
- Observe the effects on EC2 instances
- Interpret experiment results

**Prerequisites:** Module 2 completed (sandbox environment running, IAM role created)

---

## Objective

Stop one of the lab EC2 instances using FIS, verify it stops, confirm it restarts automatically, and observe the experiment lifecycle.

**Hypothesis:** "When one of our two web servers is stopped, the remaining server continues to serve traffic (simulating an Auto Scaling Group or load-balanced setup)."

---

## Option A: AWS Console

### Step 1: Create the Experiment Template

1. Open the [FIS Console](https://console.aws.amazon.com/fis)
2. Click **Create experiment template**
3. Fill in:
   - **Description**: `Lab 1: Stop a single web server EC2 instance`
   - **Name**: `lab1-ec2-stop`
   - **Account**: Use current account
4. **Actions** -- click Add action:
   - **Name**: `StopWebServer`
   - **Action type**: `aws:ec2:stop-instances`
   - **Target**: `lab1-target` (you'll create this next)
   - **Duration**: `5 minutes` (PT5M)
   - **startInstancesAfterDuration**: `true`
5. **Targets** -- click Add target:
   - **Name**: `lab1-target`
   - **Resource type**: `aws:ec2:instance`
   - **Selection mode**: `COUNT(1)`
   - **Resource tags**: Key=`Environment`, Value=`fis-lab`
   - **Resource filters**: Key=`State.Name`, Values=`running`
6. **Service access**: Select `FISExperimentRole`
7. **Stop conditions**: Skip for now (we'll add these in Module 7)
8. Click **Create experiment template**

### Step 2: Run the Experiment

1. Select the template, click **Start experiment**
2. Type `start` to confirm
3. Watch the **Timeline** tab -- you'll see the action progress through: `initiating` -> `running` -> `completed`

### Step 3: Observe

1. Open EC2 Console in another tab
2. Filter by tag `Environment=fis-lab`
3. Watch one instance transition to `stopping` -> `stopped`
4. After 5 minutes, it should return to `running`

---

## Option B: AWS CLI

### Step 1: Create the Experiment Template

Save as `lab1-template.json`:
```json
{
  "description": "Lab 1: Stop a single web server EC2 instance",
  "targets": {
    "lab1-target": {
      "resourceType": "aws:ec2:instance",
      "resourceTags": {
        "Environment": ["fis-lab"]
      },
      "filters": [
        {
          "path": "State.Name",
          "values": ["running"]
        }
      ],
      "selectionMode": "COUNT(1)"
    }
  },
  "actions": {
    "StopWebServer": {
      "actionId": "aws:ec2:stop-instances",
      "parameters": {
        "startInstancesAfterDuration": "PT5M"
      },
      "targets": {
        "Instances": "lab1-target"
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
    "Lab": "1",
    "Environment": "fis-lab"
  }
}
```

```bash
# Replace YOUR_ACCOUNT_ID in lab1-template.json, then:
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://lab1-template.json \
  --query 'experimentTemplate.id' --output text)

echo "Template created: $TEMPLATE_ID"
```

### Step 2: Start the Experiment

```bash
EXPERIMENT_ID=$(aws fis start-experiment \
  --experiment-template-id "$TEMPLATE_ID" \
  --query 'experiment.id' --output text)

echo "Experiment started: $EXPERIMENT_ID"
```

### Step 3: Monitor

```bash
# Check experiment status
aws fis get-experiment --id "$EXPERIMENT_ID" \
  --query 'experiment.state' --output json

# Watch instance states (run repeatedly or in a watch loop)
watch -n 5 "aws ec2 describe-instances \
  --filters 'Name=tag:Environment,Values=fis-lab' \
  --query 'Reservations[].Instances[].{Id:InstanceId,Name:Tags[?Key==\`Name\`].Value|[0],State:State.Name}' \
  --output table"
```

**Expected output progression:**
```
|  Id           |  Name            |  State   |
|  i-0abc123... |  fis-lab-web-a   |  running |
|  i-0def456... |  fis-lab-web-b   |  stopped |  <-- FIS stopped this one
|  i-0ghi789... |  fis-lab-app     |  running |

... after 5 minutes ...

|  i-0abc123... |  fis-lab-web-a   |  running |
|  i-0def456... |  fis-lab-web-b   |  running |  <-- auto-restarted
|  i-0ghi789... |  fis-lab-app     |  running |
```

### Step 4: Review Results

```bash
# Full experiment details
aws fis get-experiment --id "$EXPERIMENT_ID" --output json | jq '.experiment | {
  id: .id,
  state: .state,
  startTime: .startTime,
  endTime: .endTime,
  actions: .actions
}'
```

---

## Option C: Terraform

```hcl
# lab1-fis.tf

data "aws_caller_identity" "current" {}

resource "aws_fis_experiment_template" "lab1_ec2_stop" {
  description = "Lab 1: Stop a single web server EC2 instance"
  role_arn    = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/FISExperimentRole"

  stop_condition {
    source = "none"
  }

  action {
    name      = "StopWebServer"
    action_id = "aws:ec2:stop-instances"

    parameter {
      key   = "startInstancesAfterDuration"
      value = "PT5M"
    }

    target {
      key   = "Instances"
      value = "lab1-target"
    }
  }

  target {
    name           = "lab1-target"
    resource_type  = "aws:ec2:instance"
    selection_mode = "COUNT(1)"

    resource_tag {
      key    = "Environment"
      value  = "fis-lab"
    }

    filter {
      path   = "State.Name"
      values = ["running"]
    }
  }

  tags = {
    Lab         = "1"
    Environment = "fis-lab"
  }
}

output "template_id" {
  value = aws_fis_experiment_template.lab1_ec2_stop.id
}
```

```bash
terraform init && terraform apply

# Start the experiment using the template ID from output
aws fis start-experiment \
  --experiment-template-id "$(terraform output -raw template_id)"
```

> **Note**: Terraform creates the template but doesn't start experiments. You start experiments via CLI, Console, or EventBridge.

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `AccessDeniedException` when creating template | Missing `iam:PassRole` permission | Add PassRole permission for the FIS role to your user policy |
| `AccessDeniedException` during experiment | FIS role lacks `ec2:StopInstances` | Check the experiment role's permission policy |
| No instances targeted | Tag mismatch or no running instances | Verify tags: `aws ec2 describe-instances --filters "Name=tag:Environment,Values=fis-lab"` |
| Instance doesn't restart | `startInstancesAfterDuration` not set | Verify the parameter is set to `PT5M` (or any duration) |
| Experiment stays in `initiating` | IAM role propagation delay | Wait 30s and check again; new roles can take a moment |

---

## Cleanup

```bash
# Delete the experiment template
aws fis delete-experiment-template --id "$TEMPLATE_ID"

# Or with Terraform
terraform destroy
```

> The EC2 instances persist (they're part of the lab stack). Don't delete them yet -- you'll use them in the next labs.

---

## What You Learned

- How to create an experiment template (Console, CLI, Terraform)
- How `selectionMode: COUNT(1)` limits blast radius
- How `startInstancesAfterDuration` provides automatic rollback
- How to monitor experiment progress and instance state
- The experiment lifecycle: `initiating` -> `running` -> `completed`

---

## Knowledge Checkpoint

1. What happens if you set `selectionMode` to `ALL` instead of `COUNT(1)`?
2. What's the ISO 8601 format for 30 minutes?
3. If the experiment completes but the instance is still stopped, what parameter did you likely miss?
4. Can you run the same experiment template multiple times?

<details>
<summary>Answers</summary>

1. ALL matching instances with tag `Environment=fis-lab` and state `running` would be stopped -- potentially all 3 lab instances.
2. `PT30M`
3. `startInstancesAfterDuration` -- without it, FIS stops the instance but doesn't restart it.
4. Yes. Templates are reusable blueprints. Each start creates a new experiment instance.

</details>
