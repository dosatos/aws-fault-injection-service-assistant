# Module 0: Learning Paths & Glossary

**Time: 15 minutes**

---

## Learning Paths by Role

### DevOps Engineer
**Focus**: Automation, IaC, CI/CD integration

| Priority | Modules | Why |
|----------|---------|-----|
| Must do | 1, 2, 3, 7, 8 | Core experiments + automation + IaC templates |
| Should do | 4, 5 | Network and DB resilience |
| Nice to have | 6, 9 | EKS (if relevant), Game Days |

### Site Reliability Engineer (SRE)
**Focus**: Production resilience, observability, incident response

| Priority | Modules | Why |
|----------|---------|-----|
| Must do | 1, 2, 3, 5, 7, 9 | All core labs + stop conditions + production patterns |
| Should do | 4, 6, 8 | Full coverage of failure modes |
| Nice to have | 11 (capstone) | End-to-end Game Day simulation |

### Solutions Architect
**Focus**: Design patterns, architecture validation, customer guidance

| Priority | Modules | Why |
|----------|---------|-----|
| Must do | 1, 2, 3, 9 | Foundations + one hands-on lab + patterns |
| Should do | 5, 7, 8 | Multi-AZ/multi-region patterns, IaC |
| Nice to have | 4, 6 | Deep-dive labs for specific architectures |

---

## Glossary

| Term | Definition |
|------|------------|
| **Action** | A specific fault injection activity FIS performs on a resource (e.g., `aws:ec2:stop-instances`). Runs for a specified duration. |
| **Action-minute** | Billing unit. One action running for one minute = 1 action-minute ($0.10). |
| **Blast radius** | The scope of impact of an experiment. Controlled via targeting, percentages, and stop conditions. |
| **Chaos engineering** | The discipline of experimenting on a system to build confidence in its ability to withstand turbulent conditions in production. |
| **Experiment** | A running instance of an experiment template. Has a lifecycle: initiating -> running -> completed/stopped/failed. |
| **Experiment template** | A reusable blueprint that defines actions, targets, stop conditions, and the IAM role for an experiment. |
| **Game Day** | A planned event where teams intentionally inject failures to test system resilience and team response. |
| **Scenario** | A pre-built FIS template from the Scenario Library that simulates complex failure modes (e.g., AZ power interruption). |
| **Scenario Library** | AWS-curated collection of pre-built experiment scenarios in the FIS console. |
| **Steady-state hypothesis** | The expected normal behavior of your system. Chaos experiments test whether the system maintains this state under stress. |
| **Stop condition** | A CloudWatch alarm that automatically halts an experiment when a threshold is breached. Your safety net. |
| **Target** | The AWS resource(s) an action operates on. Can be specific resource IDs or tag-based selection. |

---

## Knowledge Checkpoint

Before proceeding, confirm you can answer:

1. Which learning path matches your current role?
2. What is the difference between an experiment template and an experiment?
3. What is a stop condition and why does it matter?
