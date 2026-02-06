# Module 4: Lab 2 - Network Disruption

**Time: 3 hours**

**Learning Objectives:**
- Inject network connectivity failures at the subnet level
- Inject latency and packet loss on ECS/EKS tasks (concepts transferable to EC2 with SSM)
- Understand how FIS uses NACLs to simulate network disruptions
- Test how your application handles degraded network conditions

**Prerequisites:** Module 3 completed

---

## Objective

Simulate network disruptions to test application resilience against connectivity issues. We'll cover two scenarios:

1. **Subnet connectivity blackhole** -- all traffic to/from a subnet is dropped
2. **API-level fault injection** -- simulate AWS API throttling on your IAM role

**Hypothesis:** "When network connectivity to Subnet A is disrupted, our application gracefully degrades and traffic is served from Subnet B."

---

## Lab 2A: Subnet Connectivity Disruption

This uses `aws:network:disrupt-connectivity` which creates NACL rules to deny traffic.

### How It Works Under the Hood

```
Before:                          During experiment:
+---------+                      +---------+
| Subnet  | <-- Normal NACL -->  | Subnet  | <-- DENY ALL NACL
| 10.0.1.0|     allows traffic   | 10.0.1.0|     blocks traffic
+---------+                      +---------+

FIS creates a temporary NACL with DENY rules and associates it with the target subnet.
When the experiment ends, the original NACL is restored.
```

> **Warning**: This will drop ALL traffic to/from the target subnet, including SSH. Don't target a subnet you're connected through.

### CLI Approach

Save as `lab2a-template.json`:
```json
{
  "description": "Lab 2A: Disrupt connectivity to Subnet A",
  "targets": {
    "subnet-target": {
      "resourceType": "aws:ec2:subnet",
      "resourceTags": {
        "Name": ["fis-lab-public-a"]
      },
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "DisruptSubnet": {
      "actionId": "aws:network:disrupt-connectivity",
      "parameters": {
        "duration": "PT3M",
        "scope": "all"
      },
      "targets": {
        "Subnets": "subnet-target"
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
    "Lab": "2a",
    "Environment": "fis-lab"
  }
}
```

**The `scope` parameter options:**

| Scope | Effect |
|-------|--------|
| `all` | Blocks all inbound and outbound traffic |
| `availability-zone` | Blocks traffic to/from other AZs only |
| `vpc` | Blocks traffic within VPC only |
| `dynamodb` | Blocks DynamoDB gateway endpoint traffic |
| `prefix-list` | Blocks traffic to specified prefix list |
| `s3` | Blocks S3 gateway endpoint traffic |

```bash
# Create and run
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://lab2a-template.json \
  --query 'experimentTemplate.id' --output text)

EXPERIMENT_ID=$(aws fis start-experiment \
  --experiment-template-id "$TEMPLATE_ID" \
  --query 'experiment.id' --output text)

echo "Experiment: $EXPERIMENT_ID"
```

### Monitoring During the Experiment

```bash
# Watch NACLs change (run in a separate terminal)
watch -n 5 "aws ec2 describe-network-acls \
  --filters 'Name=vpc-id,Values=YOUR_VPC_ID' \
  --query 'NetworkAcls[].{Id:NetworkAclId,Entries:Entries[?RuleAction==\`deny\`]}' \
  --output json | jq ."

# Try to ping an instance in Subnet A (should fail during experiment)
ping -c 3 INSTANCE_PUBLIC_IP

# Check experiment progress
aws fis get-experiment --id "$EXPERIMENT_ID" \
  --query 'experiment.{State:state.status,Actions:actions}' --output json
```

**Expected behavior:**
- During experiment: Instance in Subnet A unreachable, NACLs show DENY rules
- After experiment: Connectivity restored, original NACLs back

---

## Lab 2B: AWS API Fault Injection

Inject throttle errors into AWS API calls made by your application's IAM role. This tests how your app handles `Throttling` exceptions from AWS services.

Save as `lab2b-template.json`:
```json
{
  "description": "Lab 2B: Inject API throttle errors on EC2 API calls",
  "targets": {
    "iam-role-target": {
      "resourceType": "aws:iam:role",
      "resourceArns": [
        "arn:aws:iam::YOUR_ACCOUNT_ID:role/FISExperimentRole"
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "ThrottleEC2API": {
      "actionId": "aws:fis:inject-api-throttle-error",
      "parameters": {
        "duration": "PT2M",
        "service": "ec2",
        "operations": "DescribeInstances",
        "percentage": "50"
      },
      "targets": {
        "Roles": "iam-role-target"
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
    "Lab": "2b",
    "Environment": "fis-lab"
  }
}
```

**API fault injection actions:**

| Action | HTTP Error | Use Case |
|--------|-----------|----------|
| `aws:fis:inject-api-throttle-error` | 400 ThrottlingException | Test retry/backoff logic |
| `aws:fis:inject-api-internal-error` | 500 InternalError | Test error handling |
| `aws:fis:inject-api-unavailable-error` | 503 ServiceUnavailable | Test circuit breakers |

**Key parameters:**
- `service`: AWS service namespace (e.g., `ec2`, `rds`, `s3`)
- `operations`: Comma-separated API operations (e.g., `DescribeInstances,RunInstances`)
- `percentage`: 1-100, what percentage of calls get the injected error

```bash
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://lab2b-template.json \
  --query 'experimentTemplate.id' --output text)

aws fis start-experiment --experiment-template-id "$TEMPLATE_ID"

# While running, try EC2 API calls -- 50% should get throttled
for i in {1..10}; do
  aws ec2 describe-instances --query 'Reservations[0].Instances[0].InstanceId' \
    --output text 2>&1 | head -1
  sleep 1
done
```

---

## Terraform: Network Disruption

```hcl
# lab2-fis.tf

resource "aws_fis_experiment_template" "lab2_network_disruption" {
  description = "Lab 2: Disrupt subnet connectivity"
  role_arn    = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/FISExperimentRole"

  stop_condition {
    source = "none"
  }

  action {
    name      = "DisruptSubnet"
    action_id = "aws:network:disrupt-connectivity"

    parameter {
      key   = "duration"
      value = "PT3M"
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
    Lab         = "2"
    Environment = "fis-lab"
  }
}
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Experiment fails immediately | Missing NACL permissions in experiment role | Add `ec2:CreateNetworkAcl*`, `ec2:DeleteNetworkAcl*`, `ec2:ReplaceNetworkAclAssociation` to role |
| Subnet connectivity not restored after experiment | Experiment failed mid-execution (rare) | Manually check NACLs and remove FIS-created deny rules |
| API throttle injection has no effect | Wrong service name or operation | Check `service` matches the namespace (lowercase), `operations` matches exact API name |
| Can't SSH to instance | You targeted the subnet you're connected through | Use a different subnet or use Session Manager instead |

> **Pro Tip**: After any network disruption experiment, always verify NACLs are restored: `aws ec2 describe-network-acls --filters "Name=vpc-id,Values=YOUR_VPC_ID"`

---

## Cleanup

```bash
aws fis delete-experiment-template --id "$TEMPLATE_ID"
# Repeat for both 2a and 2b templates
```

---

## What You Learned

- How `aws:network:disrupt-connectivity` uses NACLs to simulate network failures
- The different `scope` options for network disruption (all, availability-zone, vpc, etc.)
- How API-level fault injection works with IAM role targeting
- The difference between infrastructure-level (NACL) and API-level fault injection
- Why you should never target the subnet you're connected through

---

## Knowledge Checkpoint

1. What AWS mechanism does FIS use to disrupt subnet connectivity?
2. What's the difference between `scope: all` and `scope: availability-zone`?
3. If you inject API throttle errors at 50%, what HTTP status code do the failed calls receive?
4. Why would you use API fault injection instead of network disruption?

<details>
<summary>Answers</summary>

1. Network ACLs (NACLs). FIS creates a temporary NACL with DENY rules and associates it with the subnet.
2. `all` blocks all traffic in/out of the subnet. `availability-zone` only blocks cross-AZ traffic, allowing intra-AZ communication.
3. HTTP 400 with `ThrottlingException` error code.
4. API fault injection is more surgical -- it targets specific API operations at a configurable percentage, simulating AWS service degradation without affecting the network layer. Network disruption is a broader blast radius.

</details>
