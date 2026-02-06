# Module 1: Foundations

**Time: 2 hours**

**Learning Objectives:**
- Understand why chaos engineering exists and when to use it
- Describe AWS FIS architecture and how it integrates with other AWS services
- Explain the 5 core concepts: experiments, actions, targets, stop conditions, templates
- Estimate costs for a given experiment

---

## 1.1 What Is Chaos Engineering and Why It Matters

Chaos engineering is the discipline of experimenting on a system to build confidence in its ability to withstand turbulent conditions in production.

**The core loop:**
```
1. Define steady state (what "normal" looks like in metrics)
2. Hypothesize that steady state continues during disruption
3. Introduce real-world failure events
4. Look for differences between control and experiment
5. Fix what broke, repeat
```

**Origin**: Netflix's Chaos Monkey (2011) randomly terminated production instances to ensure their microservices architecture could handle instance failures. This evolved into the Simian Army and the broader chaos engineering discipline.

**Why it matters for you:**
- Distributed systems fail in unexpected ways -- you can't predict all failure modes by reading code
- Testing in staging is necessary but insufficient -- production has different traffic patterns, data volumes, and race conditions
- Outages are expensive -- proactive failure testing is cheaper than reactive incident response
- Compliance and audit frameworks (SOC2, ISO 27001) increasingly require resilience testing evidence

**When NOT to do chaos engineering:**
- Your system has no monitoring or alerting (fix that first)
- You have no rollback mechanism for experiments
- You haven't done basic failure mode analysis

---

## 1.2 Introduction to AWS FIS

AWS Fault Injection Service (FIS) is a fully managed service for running controlled fault injection experiments on AWS workloads. It's part of the AWS Resilience Hub suite.

### Architecture Overview

```
+------------------+       +------------------+       +------------------+
|                  |       |                  |       |                  |
|  You (Console/   |------>|    AWS FIS       |------>|  Target AWS      |
|   CLI/SDK/IaC)   |       |   Service        |       |  Resources       |
|                  |       |                  |       |  (EC2, RDS,      |
+------------------+       +------------------+       |   EKS, etc.)     |
                                |       |             +------------------+
                                |       |
                    +-----------+       +-----------+
                    |                               |
              +-----v------+                 +------v-------+
              | CloudWatch  |                 | IAM Role     |
              | (Stop       |                 | (Permissions |
              | Conditions) |                 |  boundary)   |
              +-------------+                 +--------------+
```

**How it works:**
1. You create an **experiment template** defining what to break, how, and for how long
2. You **start an experiment** from that template
3. FIS **assumes an IAM role** you provide to get permissions on your resources
4. FIS **executes actions** on targets (stop instances, inject latency, etc.)
5. FIS monitors **stop conditions** (CloudWatch alarms) and aborts if thresholds breach
6. Actions either complete naturally or are rolled back when the experiment ends

**Key differentiators vs. open-source tools (Chaos Monkey, Litmus, ChaosMesh):**
- No agents to install or maintain (except for some ECS/EKS stress tests)
- Native AWS API integration -- uses the same control plane your services use
- Built-in safety via stop conditions and IAM permission boundaries
- Pre-built Scenario Library for complex multi-service failure simulations
- Audit trail via CloudTrail

---

## 1.3 Core Concepts

### Experiment Templates
A reusable JSON definition containing:
- **Description**: What this experiment tests
- **Actions**: What faults to inject (with sequencing)
- **Targets**: Which resources to affect
- **Stop conditions**: Safety guardrails
- **Role ARN**: IAM role FIS assumes
- **Tags**: For organization and cost tracking

### Actions
What FIS does to your resources. Actions can run **sequentially** or **in parallel**.

Each action has:
- A **type** (e.g., `aws:ec2:stop-instances`)
- A **duration** (ISO 8601 format, e.g., `PT5M` = 5 minutes)
- **Parameters** (action-specific, e.g., `startInstancesAfterDuration: true`)
- A **target** reference

**Action categories:**

| Category | Examples |
|----------|----------|
| Compute | Stop/terminate/reboot EC2, spot interruption |
| Network | Disrupt connectivity, inject latency/packet loss |
| Database | RDS failover, reboot DB instances |
| Containers | ECS task stop/stress, EKS pod delete/stress |
| Storage | Pause EBS I/O, inject EBS latency |
| Serverless | Lambda delay, invocation errors |
| API-level | Inject throttling, 500s, 503s on AWS API calls |
| Replication | Pause DynamoDB/S3/MemoryDB replication |

### Targets
The resources an action operates on. Two selection modes:

| Mode | When to Use |
|------|-------------|
| **Specific resources** (`resourceArns`) | Testing a known critical instance |
| **Tag-based** (`resourceTags` + `filters`) | Testing a class of resources (e.g., all `env:staging` instances) |

Additional controls:
- **selectionMode**: `ALL`, `COUNT(n)`, or `PERCENT(n)` -- controls blast radius
- **filters**: Narrow by state (e.g., only running instances)

### Stop Conditions
CloudWatch alarms that act as circuit breakers. If any stop condition alarm enters `ALARM` state, FIS immediately halts the experiment and rolls back reversible actions.

Examples:
- HTTP 5xx error rate > 5%
- Latency p99 > 2 seconds
- CPU utilization > 95% on critical service

### Scenario Library
Pre-built, AWS-curated experiment scenarios available in the FIS console:
- **AZ Availability: Power Interruption** -- simulates complete AZ power loss
- **Cross-Region Connectivity Disruption** -- blocks cross-region traffic
- **Application Slowdown in AZ** -- injects latency in a single AZ

These combine multiple actions into realistic failure scenarios without you having to build them from scratch.

---

## 1.4 Pricing

| Component | Cost | Notes |
|-----------|------|-------|
| Action-minute | $0.10 | Per action, per minute, rounded up |
| Multi-account surcharge | +$0.10/action-min per additional account | Only for multi-account experiments |
| Experiment report | $5.00 per report | Optional resilience documentation |
| GovCloud | $0.12/action-min | Higher rate for GovCloud regions |

**Cost examples:**

| Scenario | Calculation | Cost |
|----------|-------------|------|
| Stop 1 EC2 for 5 min | 1 action x 5 min x $0.10 | $0.50 |
| 2 parallel actions, 20 min each + 1 sequential 10 min | (2x20 + 1x10) x $0.10 | $5.00 |
| 2 actions, 20 min, across 5 accounts | 2x20 x $0.10 x 5 | $20.00 |
| Full Game Day: 10 actions, avg 15 min | 10x15 x $0.10 | $15.00 |

> **Pro Tip**: FIS itself is cheap. The real cost is the resources you run for testing (EC2 instances, RDS clusters, EKS nodes). Use `t3.micro`/`t3.small` instances for labs and tear down immediately after.

---

## 1.5 How FIS Integrates with Other AWS Services

```
CloudTrail  <-- Audit logs for all FIS API calls
CloudWatch  <-- Stop conditions + experiment monitoring
IAM         <-- Permission boundaries for experiments
EventBridge <-- Trigger experiments on schedule or events
Lambda      <-- Automate pre/post experiment actions
SSM         <-- Custom actions via Run Command / Automation
Resilience Hub <-- Recommendations feed into FIS experiments
```

---

## Knowledge Checkpoint

1. What are the 4 steps of the chaos engineering loop?
2. Name 3 action categories in FIS and give an example action for each.
3. What is the difference between `COUNT(n)` and `PERCENT(n)` selection modes?
4. An experiment runs 3 actions in parallel for 10 minutes each. What's the cost?
5. Why are stop conditions critical for production experiments?

<details>
<summary>Answers</summary>

1. Define steady state, hypothesize continuity, inject failure, observe differences (then fix and repeat)
2. E.g., Compute: `aws:ec2:stop-instances`, Network: `aws:network:disrupt-connectivity`, Database: `aws:rds:failover-db-cluster`
3. `COUNT(n)` targets exactly n resources; `PERCENT(n)` targets n% of matching resources
4. 3 actions x 10 min x $0.10 = $3.00
5. They act as circuit breakers to prevent experiments from causing unacceptable damage -- they automatically stop the experiment if a CloudWatch alarm triggers

</details>
