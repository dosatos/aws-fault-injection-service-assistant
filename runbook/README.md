# AWS Fault Injection Service (FIS) - Self-Study Runbook

A progressive, hands-on learning path for mastering AWS FIS. Takes you from zero to confidently running chaos experiments in production within 2-4 weeks.

---

## Modules

| # | Module | Time | Description |
|---|--------|------|-------------|
| 0 | [Learning Paths & Glossary](00-learning-paths.md) | 15 min | Role-based paths, milestones, key terms |
| 1 | [Foundations](01-foundations.md) | 2 hrs | Chaos engineering principles, FIS architecture, pricing |
| 2 | [Prerequisites & Setup](02-prerequisites-setup.md) | 3 hrs | IAM, sandbox environment, tooling |
| 3 | [Lab: EC2 Stop/Start](03-lab-ec2-stop-start.md) | 2 hrs | Your first experiment |
| 4 | [Lab: Network Disruption](04-lab-network-disruption.md) | 3 hrs | Latency, packet loss, connectivity blackholes |
| 5 | [Lab: RDS Failover](05-lab-rds-failover.md) | 2 hrs | Database resilience testing |
| 6 | [Lab: EKS Pod Failures](06-lab-eks-pod-failures.md) | 3 hrs | Kubernetes chaos experiments |
| 7 | [Lab: Multi-Service + Stop Conditions](07-lab-multi-service-stop-conditions.md) | 3 hrs | Complex experiments with safety guardrails |
| -- | [Lab: ECS SQS Cross-Account Processor](lab-ecs-sqs-crossaccount.md) | 3-4 hrs | Fargate, SQS FIFO, cross-account DDB, Security Hub API faults |
| 8 | [Advanced Topics](08-advanced-topics.md) | 4 hrs | Custom actions, IaC, automation, observability |
| 9 | [Real-World Patterns](09-real-world-patterns.md) | 3 hrs | Game Days, production safety, culture |
| 10 | [Reference](10-reference.md) | - | Actions cheat sheet, CLI commands, links |
| 11 | [Appendix](11-appendix.md) | 4 hrs | FAQ, capstone project |

**Total estimated time: 29 hours (~2-4 weeks at 2 hrs/day)**

---

## Milestone Tracker

### Day 1 (Modules 0-1)
- [ ] Understand chaos engineering principles
- [ ] Know the 5 core FIS concepts (experiments, actions, targets, stop conditions, templates)
- [ ] Estimate costs for a basic experiment

### Week 1 (Modules 2-5)
- [ ] Sandbox environment running
- [ ] Completed 3 hands-on labs (EC2, network, RDS)
- [ ] Can write experiment templates in JSON and Terraform

### Month 1 (Modules 6-11)
- [ ] Completed all labs including EKS and multi-service
- [ ] Can design custom experiments with SSM documents
- [ ] Planned and executed a Game Day
- [ ] Completed the capstone project

---

## How to Use This Runbook

1. **Start with Module 0** to pick the learning path for your role
2. **Work sequentially** -- each module builds on the previous
3. **Do every lab** -- reading without doing won't stick
4. **Answer the checkpoint questions** before moving to the next module
5. **Use Module 10** as your daily reference once you're past the labs

> **Cost Warning**: Running labs will incur AWS charges. Budget ~$20-50 for the full runbook if you clean up promptly after each lab. Module 2 includes a cleanup checklist.
