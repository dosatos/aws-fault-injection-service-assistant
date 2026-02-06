# AWS Fault Injection Service Assistant

An AI-powered chaos engineering assistant backed by a comprehensive AWS FIS runbook. Open this repo with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and get guided experiment creation, template reviews, and Game Day planning -- all grounded in curated best practices.

## Quick Start

```bash
git clone https://github.com/dosatos/aws-fault-injection-service-assistant.git
cd aws-fault-injection-service-assistant
claude
```

Then use the built-in skills:

```
/create-experiment           # Guided experiment template builder
/create-experiment rds-failover   # Start with a specific action type
/review-experiment template.json  # Audit a template for safety issues
/game-day                    # Plan a Game Day from scratch
```

Or just ask questions -- Claude has the full runbook as context:

```
> What FIS actions are available for ECS Fargate?
> How do I set up IAM for cross-account experiments?
> What stop conditions should I use for a production experiment?
```

## What's Inside

### Runbook (`runbook/`)

A progressive self-study path covering AWS FIS from beginner to proficient. 14 modules, ~4700 lines, estimated 2-4 weeks at 2 hrs/day.

| Module | Topic |
|--------|-------|
| [Foundations](runbook/01-foundations.md) | Chaos engineering principles, FIS architecture, pricing |
| [Prerequisites & Setup](runbook/02-prerequisites-setup.md) | IAM roles, sandbox CloudFormation, tooling |
| [Lab: EC2 Stop/Start](runbook/03-lab-ec2-stop-start.md) | First experiment (Console + CLI + Terraform) |
| [Lab: Network Disruption](runbook/04-lab-network-disruption.md) | Subnet connectivity, API fault injection |
| [Lab: RDS Failover](runbook/05-lab-rds-failover.md) | Aurora cluster failover, measuring recovery time |
| [Lab: EKS Pod Failures](runbook/06-lab-eks-pod-failures.md) | Pod delete, CPU stress, network latency |
| [Lab: Multi-Service](runbook/07-lab-multi-service-stop-conditions.md) | Parallel/sequential actions, stop conditions |
| [Lab: ECS SQS Cross-Account](runbook/lab-ecs-sqs-crossaccount.md) | Fargate, FIFO SQS, cross-account DDB, Security Hub |
| [Advanced Topics](runbook/08-advanced-topics.md) | SSM custom actions, IaC, EventBridge automation |
| [Real-World Patterns](runbook/09-real-world-patterns.md) | Game Days, production safety, resilience KPIs |
| [Reference](runbook/10-reference.md) | All ~45 FIS actions, CLI cheat sheet, links |
| [Appendix](runbook/11-appendix.md) | FAQ, capstone project, sample templates |

See [runbook/README.md](runbook/README.md) for the full table of contents and milestone tracker.

### Claude Code Configuration (`.claude/`)

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Project context, core rules (safety, IAM scoping, pricing), runbook cross-references |
| `.claude/CLAUDE.md` | Detailed domain rules: environment-based safety matrix, mandatory warnings, blast radius guidance |
| `.claude/skills/create-experiment/` | `/create-experiment` -- 6-step guided workflow: failure type, targets, environment, JSON + Terraform output |
| `.claude/skills/review-experiment/` | `/review-experiment` -- 8-category audit: action validity, target safety, stop conditions, reversibility, IAM, cost |
| `.claude/skills/game-day/` | `/game-day` -- Architecture analysis, failure scenarios, hypotheses, experiment templates, monitoring, checklists |
| `.claude/agents/fis-advisor.md` | Read-only domain expert subagent that looks up answers in the runbook |

## Skills in Detail

### `/create-experiment`

Interactive workflow that walks you through designing a new FIS experiment:

1. **Failure type** -- maps your intent to valid FIS actions
2. **Targets** -- resource selection with tag-based scoping
3. **Environment** -- applies safety rules (dev: relaxed, staging: stop conditions required, prod: strict blast radius)
4. **Output** -- generates both JSON and Terraform templates with cost estimates
5. **Review** -- validates action names, stop conditions, reversibility
6. **Next steps** -- points to the matching runbook lab

### `/review-experiment`

Audits an existing experiment template (JSON or HCL) against 8 categories:

- Action validity (cross-checks against official action list)
- Target safety (`ALL` in production = FAIL)
- Stop conditions (missing in prod = FAIL)
- Reversibility (`stop-instances` without `startInstancesAfterDuration` = WARN)
- IAM role scope
- Action sequencing logic
- Tags and metadata
- Cost estimate

### `/game-day`

Plans a complete Game Day:

- Analyzes your architecture and proposes failure scenarios
- Writes hypotheses for each experiment
- Generates ordered experiment templates
- Sets up monitoring dashboards
- Produces a ready-to-use Game Day document with pre/post checklists

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- AWS account (for running experiments)
- AWS CLI v2, Terraform (optional), jq

## Cost

- **FIS**: $0.10 per action-minute. A typical lab costs $0.50-5.00.
- **Lab resources**: EC2 `t3.micro` ~$0.01/hr, Aurora `db.t3.medium` ~$0.07/hr. Budget $20-50 for the full runbook.
- Tear down resources after each lab session. Cleanup instructions are included in every module.

## License

This project is provided as-is for educational purposes.
