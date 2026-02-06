---
name: game-day
description: Plan an AWS FIS Game Day — from architecture analysis to experiment templates, hypotheses, monitoring setup, and team checklists.
argument-hint: "[optional: architecture description or service name]"
---

# Plan a Game Day

You are helping the user plan and prepare a chaos engineering Game Day using AWS FIS. Reference @runbook/09-real-world-patterns.md for the checklist framework and safety guidelines.

## Step 1: Understand the Architecture

If `$ARGUMENTS` is provided, use it as a starting point. Otherwise, ask the user to describe:

1. **What system are you testing?** (service name, purpose)
2. **Key components?** (EC2/ECS/EKS, databases, queues, caches, load balancers)
3. **Multi-AZ or multi-region?**
4. **Dependencies?** (cross-account, third-party APIs, shared services)
5. **Current monitoring?** (CloudWatch alarms, dashboards, APM tools)

Draw an ASCII architecture diagram based on their description.

## Step 2: Identify Failure Scenarios

Based on the architecture, propose 3-5 realistic failure scenarios. For each scenario, reference the matching runbook module:

Map architecture components to experiment types:
- EC2 / ASG → Instance stop, spot interruption (@runbook/03-lab-ec2-stop-start.md)
- VPC / Subnets → Network disruption (@runbook/04-lab-network-disruption.md)
- RDS / Aurora → Cluster failover (@runbook/05-lab-rds-failover.md)
- EKS → Pod deletion, CPU stress, network latency (@runbook/06-lab-eks-pod-failures.md)
- ECS + SQS → Task kill, API throttling (@runbook/lab-ecs-sqs-crossaccount.md)
- Cross-account → STS throttling, DDB throttling (@runbook/lab-ecs-sqs-crossaccount.md)
- Serverless → Lambda delay/errors (reference @runbook/10-reference.md Lambda actions)
- Multi-AZ → Use FIS Scenario Library "AZ Availability: Power Interruption"
- AWS API dependency → API fault injection (throttle/500/503)

Prioritize scenarios by:
1. Likelihood of occurring in production
2. Impact if not handled properly
3. Whether the team has tested this before (never tested = higher priority)

## Step 3: Write Hypotheses

For each scenario, write a hypothesis following this format:

> **Hypothesis**: "When [specific failure], [system/component] will [expected behavior], and [user-facing impact] will be [acceptable level]."

Example:
> "When the primary Aurora writer fails over, our API will reconnect within 30 seconds and error rate will stay below 1%."

## Step 4: Design Experiment Templates

For each scenario, generate:
1. FIS experiment template (JSON)
2. Appropriate stop conditions (CloudWatch alarms)
3. Expected duration

Apply safety rules from @.claude/CLAUDE.md:
- Start with `COUNT(1)` for first Game Day
- Always include stop conditions
- Keep durations ≤ 10 min per experiment
- Leave 5-10 min observation gaps between experiments

Order experiments from lowest risk to highest risk.

## Step 5: Prepare Monitoring

For each experiment, suggest:
1. CloudWatch metrics to watch
2. Dashboard widgets (provide `aws cloudwatch put-dashboard` commands)
3. Log groups to tail
4. Application-level health checks

## Step 6: Generate Game Day Document

Output a complete Game Day document using this structure (from @runbook/09-real-world-patterns.md):

```markdown
# Game Day: [Name]
**Date**: [suggest next Tuesday during business hours]
**Estimated Duration**: [sum of experiments + observation gaps]
**Participants**: [suggest roles: facilitator, safety officer, observers]
**System Under Test**: [from Step 1]

## Architecture
[ASCII diagram from Step 1]

## Pre-Game Checklist
- [ ] All experiment templates created and tested in dev/staging
- [ ] Stop conditions verified (manually triggered to confirm they work)
- [ ] Monitoring dashboards open and confirmed working
- [ ] War room / Slack channel created
- [ ] On-call engineer briefed
- [ ] Rollback procedures documented for each experiment
- [ ] Stakeholders notified (engineering managers, customer support)

## Experiments (in order)

### Experiment 1: [Name]
**Hypothesis**: [from Step 3]
**Template ID**: [placeholder]
**Duration**: [X min]
**Stop Condition**: [alarm name]
**Success Criteria**:
- [ ] [specific measurable criteria]
**Observation Gap**: 5 min

[repeat for each experiment]

## Abort Criteria
- Any stop condition alarm triggered
- Error rate exceeds [X]%
- Team consensus to stop
- Unexpected resource affected

## Post-Game
- [ ] All experiments completed or stopped
- [ ] All resources verified in healthy state
- [ ] NACLs restored (if network experiments were run)
- [ ] Findings documented
- [ ] Retrospective scheduled within 48 hours

## Findings Template
| # | Experiment | Finding | Severity | Action Item | Owner | Deadline |
|---|-----------|---------|----------|-------------|-------|----------|
```

## Step 7: Cleanup Commands

Provide a cleanup script that:
1. Deletes all experiment templates created for the Game Day
2. Deletes CloudWatch alarms/dashboards
3. Verifies no experiments are still running
4. Checks for leftover NACL rules
