---
name: create-experiment
description: Guided workflow to design a new AWS FIS experiment template. Walks through failure type, targets, environment, and safety — then generates JSON and Terraform.
argument-hint: "[optional: action type or service, e.g. 'ec2-stop' or 'rds-failover']"
---

# Create FIS Experiment Template

You are building an AWS FIS experiment template interactively. Follow these steps strictly.

## Step 1: Understand the Failure Scenario

If `$ARGUMENTS` was provided, use it as a starting hint. Otherwise, ask the user:

**What failure do you want to simulate?** Map their answer to FIS action categories. Look up valid actions in @runbook/10-reference.md.

Common mappings:
- "stop instance" / "instance failure" → `aws:ec2:stop-instances`
- "network issue" / "latency" → `aws:network:disrupt-connectivity` or `aws:ecs:task-network-latency`
- "database failover" → `aws:rds:failover-db-cluster`
- "pod failure" → `aws:eks:pod-delete`
- "API errors" / "throttling" → `aws:fis:inject-api-throttle-error`
- "Lambda" → `aws:lambda:invocation-add-delay` or `aws:lambda:invocation-error`
- "ECS task" → `aws:ecs:stop-task`
- "spot interruption" → `aws:ec2:send-spot-instance-interruptions`

If the user describes a scenario that maps to multiple actions (e.g., "AZ outage"), design a multi-action experiment with `startAfter` sequencing. Reference @runbook/07-lab-multi-service-stop-conditions.md for patterns.

## Step 2: Define Targets

Ask the user:
1. **What resources should be targeted?** (specific ARNs or tag-based selection?)
2. **What tags identify them?** (e.g., `Environment=staging`, `Team=payments`)
3. **How many to affect?** → Map to `selectionMode`: `COUNT(n)`, `PERCENT(n)`, or `ALL`

Apply these rules:
- Production → suggest `COUNT(1)` first, then `PERCENT(10)` max
- Staging → `PERCENT(25)` is reasonable
- Dev → `ALL` is acceptable

If targeting EKS pods, remind about Kubernetes RBAC setup (reference @runbook/06-lab-eks-pod-failures.md section 6.1).
If targeting cross-account resources, remind about AssumeRole setup (reference @runbook/lab-ecs-sqs-crossaccount.md).

## Step 3: Determine Environment and Safety

Ask: **What environment is this for?** (dev / staging / production)

Apply the safety rules from @.claude/CLAUDE.md:
- **Dev**: Stop conditions optional, any selection mode
- **Staging**: At least 1 CloudWatch alarm stop condition required
- **Production**: Multiple stop conditions required, `COUNT(1)` or `PERCENT(≤10)`, duration ≤ 10 min

If staging/production, ask: **What CloudWatch alarm(s) should act as stop conditions?** Suggest:
- HTTP 5xx error rate alarm
- Latency p99 alarm
- Application-specific health check alarm

If they don't have alarms yet, provide the `aws cloudwatch put-metric-alarm` command to create one.

## Step 4: Generate the Template

Produce both JSON and Terraform formats:

1. **JSON template** — ready for `aws fis create-experiment-template --cli-input-json file://template.json`
2. **Terraform template** — `aws_fis_experiment_template` resource

Include in comments:
- Cost estimate: `# Cost: [N] actions × [M] min × $0.10 = $X.XX`
- Blast radius: `# Blast radius: [selectionMode] of [resourceType] with tags [tags]`
- Recommended runbook lab for the chosen action type

## Step 5: Review Checklist

Before presenting the final template, verify:
- [ ] All action names are valid (cross-check with @runbook/10-reference.md)
- [ ] Targets have explicit tags or ARNs (no unscoped targets)
- [ ] Stop conditions present (if staging/prod)
- [ ] Duration uses ISO 8601 format (e.g., `PT5M`)
- [ ] `startInstancesAfterDuration` set on `stop-instances` actions
- [ ] Tags include at minimum: `Environment`, `Team`
- [ ] IAM role ARN placeholder included with a note to scope it properly

## Step 6: Suggest Next Steps

After generating the template:
1. Point to the matching runbook lab for a hands-on walkthrough
2. Suggest creating the experiment in a dev/staging environment first
3. Remind about the blast radius progression: `COUNT(1)` → `PERCENT(10)` → `PERCENT(25)`
4. If this is their first experiment, point to @runbook/02-prerequisites-setup.md for IAM and sandbox setup
