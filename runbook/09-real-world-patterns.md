# Module 9: Real-World Patterns

**Time: 3 hours**

**Learning Objectives:**
- Plan and execute a Game Day
- Establish production experiment safety guidelines
- Design experiments for different architectures
- Define metrics and KPIs to measure resilience improvements

---

## 9.1 Game Day Planning and Execution

A Game Day is a structured event where teams intentionally inject failures to test both system resilience AND team response.

### Game Day Checklist

**2 Weeks Before:**
- [ ] Define the scope: which system(s) and failure scenarios
- [ ] Write hypotheses: "We believe X will happen when Y fails"
- [ ] Create and test experiment templates in staging
- [ ] Notify stakeholders (SRE, on-call, engineering managers, customer support)
- [ ] Verify monitoring dashboards cover all affected services
- [ ] Confirm rollback procedures are documented and tested
- [ ] Schedule during low-traffic window (first time only)

**1 Day Before:**
- [ ] Verify all stop conditions are configured and working
- [ ] Confirm the on-call engineer is aware and available
- [ ] Pre-create a dedicated Slack channel / war room
- [ ] Review and dry-run the experiment in staging one more time
- [ ] Brief the team: what will happen, who does what, how to abort

**During:**
- [ ] Announce start in the war room
- [ ] Start recording: who observes what, timestamps, screenshots
- [ ] Run experiments one at a time (for first Game Day)
- [ ] Document unexpected behaviors immediately
- [ ] Have a designated "safety officer" who can stop the experiment
- [ ] Monitor stop conditions actively

**After:**
- [ ] Run the full cleanup checklist
- [ ] Hold a blameless retrospective within 48 hours
- [ ] Document findings: what broke, what held, what was unexpected
- [ ] Create action items with owners and deadlines
- [ ] Update runbooks based on findings
- [ ] Schedule the next Game Day

### Game Day Template Document

```markdown
# Game Day: [Name]
**Date**: YYYY-MM-DD
**Participants**: [names]
**System Under Test**: [service/application name]

## Hypotheses
1. When [failure], [expected behavior]
2. When [failure], [expected behavior]

## Experiments (in order)
| # | Experiment | Template ID | Duration | Stop Condition |
|---|-----------|-------------|----------|----------------|
| 1 | ...       | EXT...      | 5 min    | ErrorRate > 5% |

## Success Criteria
- [ ] Error rate stays below X%
- [ ] Latency p99 stays below Xms
- [ ] No data loss
- [ ] Auto-recovery within X minutes

## Abort Criteria
- Error rate exceeds X%
- Any manual stop condition triggered
- Unexpected resource affected

## Findings
| # | Finding | Severity | Action Item | Owner |
|---|---------|----------|-------------|-------|

## Follow-up
- Next Game Day date: ___
- Action items deadline: ___
```

---

## 9.2 Production Experiment Safety Guidelines

### The Safety Pyramid

```
                    /\
                   /  \
                  / P  \          Production (full traffic)
                 / R  O \         - Multiple stop conditions
                / O  D   \       - PERCENT(5-10%) targets
               /----------\      - Business hours, team on standby
              /  STAGING    \     Staging (synthetic traffic)
             /   Full scope  \    - Tag-based isolation
            /   experiments   \   - Full experiment coverage
           /------------------\
          /     DEV / LOCAL     \  Development
         /   Learn the tools,   \ - Basic experiments
        /    validate templates  \ - No stop conditions needed
       /________________________\
```

### Production Safety Rules

| Rule | Why | How |
|------|-----|-----|
| **Start small** | Limit blast radius | `PERCENT(5)` or `COUNT(1)` first, increase gradually |
| **Always use stop conditions** | Automated circuit breaker | Alarm on error rate, latency, key business metrics |
| **Tag resources for opt-in** | Prevent accidental targeting | `ChaosReady: true` tag required in target filter |
| **Run during business hours** | Team available to respond | Avoid nights/weekends for first experiments |
| **Notify stakeholders** | No surprises | Slack announcement + calendar invite for Game Days |
| **Have a rollback plan** | Quick recovery if things go wrong | Document manual rollback steps for each experiment |
| **One change at a time** | Isolate cause and effect | Don't run multiple experiments simultaneously (initially) |
| **Monitor the experiment** | Real-time awareness | Dedicated dashboard open during experiment |
| **Time-box experiments** | Limit exposure | Use short durations (5-15 min) in production |

### Blast Radius Progression

```
Week 1:  COUNT(1) in dev         → Learn the mechanics
Week 2:  PERCENT(10) in staging  → Validate hypotheses
Week 3:  COUNT(1) in production  → First real test
Week 4:  PERCENT(10) in prod     → Broader validation
Month 2: PERCENT(25) in prod     → Confidence building
Month 3: AZ-level scenarios      → Full resilience validation
```

---

## 9.3 Architecture-Specific Experiment Scenarios

### Microservices Architecture

| Scenario | FIS Actions | What You Learn |
|----------|------------|----------------|
| Service dependency failure | `aws:ecs:stop-task` on dependency service | Circuit breaker behavior, graceful degradation |
| Network partition between services | `aws:network:disrupt-connectivity` on service subnet | Retry logic, timeout handling |
| Slow dependency | `aws:ecs:task-network-latency` on dependency | Cascade failure risk, timeout settings |
| Database failover | `aws:rds:failover-db-cluster` | Connection pool recovery, query retry logic |
| Cache failure | `aws:elasticache:replicationgroup-interrupt-az-power` | Cache miss handling, database load spike |

### Serverless Architecture

| Scenario | FIS Actions | What You Learn |
|----------|------------|----------------|
| Lambda cold start spike | `aws:lambda:invocation-add-delay` | Client timeout handling |
| Lambda errors | `aws:lambda:invocation-error` | Dead letter queue processing, error handling |
| DynamoDB throttling | `aws:fis:inject-api-throttle-error` targeting DynamoDB | Retry logic, capacity planning |
| API Gateway dependency failure | Combine Lambda errors + API throttling | End-to-end error propagation |

### Multi-AZ / Multi-Region Architecture

| Scenario | FIS Actions | What You Learn |
|----------|------------|----------------|
| AZ power outage | Use Scenario Library: "AZ Availability: Power Interruption" | Full AZ failover behavior |
| Cross-region connectivity loss | `aws:network:transit-gateway-disrupt-cross-region-connectivity` | Regional failover, DNS failover timing |
| DynamoDB Global Table replication pause | `aws:dynamodb:global-table-pause-replication` | Eventual consistency behavior, conflict resolution |
| S3 cross-region replication pause | `aws:s3:bucket-pause-replication` | Backup/DR data staleness |

---

## 9.4 Measuring Resilience: Metrics and KPIs

### Core Resilience Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **MTTR** (Mean Time to Recovery) | Average time from failure detection to full recovery | < 5 min for critical services |
| **MTTD** (Mean Time to Detect) | Average time from failure occurrence to detection | < 1 min with proper monitoring |
| **Blast Radius Impact** | % of users/requests affected during experiment | Decreasing over time |
| **Recovery Automation Rate** | % of failures that self-heal without human intervention | > 90% |
| **Error Budget Burn** | How much error budget the experiment consumed | < 10% of monthly budget |

### Tracking Improvement Over Time

```
Experiment Log Template:
┌─────────────┬──────────┬──────┬───────────┬──────────────┐
│ Date        │ Scenario │ MTTD │ MTTR      │ Users Impact │
├─────────────┼──────────┼──────┼───────────┼──────────────┤
│ 2025-01-15  │ AZ fail  │ 45s  │ 8 min     │ 12%          │
│ 2025-02-15  │ AZ fail  │ 30s  │ 4 min     │ 5%           │ ← improvement!
│ 2025-03-15  │ AZ fail  │ 15s  │ 90s       │ 2%           │ ← improvement!
└─────────────┴──────────┴──────┴───────────┴──────────────┘
```

### Building a Resilience Scorecard

Rate each service dimension on a 1-5 scale:

| Dimension | 1 (Poor) | 3 (Adequate) | 5 (Excellent) |
|-----------|----------|-------------|---------------|
| **Redundancy** | Single point of failure | Multi-AZ | Multi-region active-active |
| **Auto-recovery** | Manual intervention required | Partial automation | Fully automated self-healing |
| **Monitoring** | No alerting | Basic metrics | Full observability + anomaly detection |
| **Chaos testing** | Never tested | Tested in staging | Regular production experiments |
| **Runbooks** | No documentation | Basic steps | Automated runbooks |

---

## 9.5 Building a Chaos Engineering Culture

### Maturity Model

| Level | Description | Activities |
|-------|------------|-----------|
| **1. Ad-hoc** | Manual, one-off experiments | First Game Day, learning FIS basics |
| **2. Repeatable** | Documented experiments, templates in IaC | Monthly Game Days, staging experiments automated |
| **3. Defined** | Standard process, metrics tracked | Production experiments, resilience scorecard |
| **4. Measured** | Data-driven decisions, trend analysis | Continuous experiments in CI/CD, SLO-driven |
| **5. Optimized** | Chaos engineering is routine | Automated regression experiments, self-healing validated continuously |

### Getting Buy-In

| Audience | Message |
|----------|---------|
| **Engineering managers** | "We found 3 critical bugs in staging that would have caused production outages" |
| **Executives** | "Our MTTR improved from 15 min to 2 min after 3 months of chaos testing, preventing an estimated $X in downtime costs" |
| **Developers** | "This catches the bugs that unit tests and integration tests miss -- real infrastructure failures" |
| **Security/Compliance** | "FIS experiments provide auditable evidence of resilience testing for SOC2/ISO compliance" |

---

## Knowledge Checkpoint

1. What is the first thing you should do if a production experiment causes unexpected behavior?
2. What selection mode should you start with in production?
3. Name 3 metrics you'd track to measure resilience improvement over time.
4. Why should you run first production experiments during business hours?

<details>
<summary>Answers</summary>

1. Stop the experiment immediately (via console, CLI `aws fis stop-experiment`, or let the stop condition trigger). Then investigate.
2. `COUNT(1)` -- affect a single resource first to minimize blast radius.
3. MTTR, MTTD, blast radius impact (% users affected), recovery automation rate, error budget burn.
4. Because the team is available to respond, observe, and intervene if something goes wrong. Night/weekend experiments are fine once you're confident in your stop conditions and auto-recovery.

</details>
