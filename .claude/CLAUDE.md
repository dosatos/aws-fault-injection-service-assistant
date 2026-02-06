# FIS Assistant — Detailed Domain Rules

## Pricing Awareness
- FIS costs $0.10 per action-minute (standard regions), $0.12 in GovCloud
- Multi-account experiments: +$0.10/action-min per additional target account
- Reports: $5.00 each
- Always include a cost estimate when generating experiment templates

## Safety Enforcement

### By Environment
| Environment | Stop Conditions | Selection Mode | Duration |
|-------------|----------------|----------------|----------|
| dev/lab | Optional (`"source": "none"` OK) | Any | Any |
| staging | Required (at least 1 CloudWatch alarm) | Recommend PERCENT or COUNT | ≤ 15 min |
| production | Required (multiple alarms recommended) | Must use COUNT(1) first, then PERCENT(≤10) | ≤ 10 min |

### Mandatory Warnings
Emit a ⚠️ WARNING when any of these are present:
- `aws:ec2:terminate-instances` without confirmation this is intentional
- `selectionMode: ALL` targeting production-tagged resources
- `aws:ec2:stop-instances` without `startInstancesAfterDuration` parameter
- No `stopConditions` on any non-dev experiment
- `"Resource": "*"` in IAM policy for production roles
- Network disruption (`aws:network:disrupt-connectivity`) targeting a subnet the user may be connected through

### Blast Radius Guidance
- First experiment in any new environment: `COUNT(1)`
- Second run: `PERCENT(10)`
- Only escalate after validating stop conditions work
- Reference: runbook/09-real-world-patterns.md (Safety Pyramid)

## Template Structure Rules
Every experiment template must have:
1. A descriptive `description` field (what + why)
2. At least one `target` with explicit `resourceTags` or `resourceArns`
3. At least one `action` with a `duration` parameter
4. A `roleArn` pointing to a scoped FIS experiment role
5. `stopConditions` (unless explicitly lab/dev)
6. `tags` for tracking (Environment, Team, Scenario at minimum)

## Cross-Reference Map
When the user asks about a topic, read the relevant runbook file:
- "How does FIS work?" → runbook/01-foundations.md
- "Set up IAM" or "permissions" → runbook/02-prerequisites-setup.md
- "EC2" or "stop instance" → runbook/03-lab-ec2-stop-start.md
- "network" or "latency" or "packet loss" or "NACL" → runbook/04-lab-network-disruption.md
- "RDS" or "Aurora" or "failover" or "database" → runbook/05-lab-rds-failover.md
- "EKS" or "pod" or "Kubernetes" → runbook/06-lab-eks-pod-failures.md
- "stop condition" or "multi-service" or "parallel" or "sequential" → runbook/07-lab-multi-service-stop-conditions.md
- "ECS" or "SQS" or "cross-account" or "Fargate" → runbook/lab-ecs-sqs-crossaccount.md
- "SSM" or "custom action" or "Terraform" or "CloudFormation" or "EventBridge" → runbook/08-advanced-topics.md
- "Game Day" or "production" or "safety" or "culture" or "metrics" → runbook/09-real-world-patterns.md
- Action name lookup, CLI commands, ISO 8601 → runbook/10-reference.md
- FAQ, sample templates → runbook/11-appendix.md

## Output Format
- Use code blocks with `json` or `hcl` language tags for templates
- Include `# Cost estimate: X actions × Y min × $0.10 = $Z.ZZ` as a comment
- Include `# Blast radius: COUNT(1) of [resource type] in [scope]` as a comment
- After generating a template, suggest the matching runbook lab for a walkthrough
