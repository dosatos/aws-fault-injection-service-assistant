# Lab: ECS Fargate SQS Processor - Cross-Account Resilience Testing

**Time: 3-4 hours**

**Learning Objectives:**
- Test resilience of an async message processor running on ECS Fargate
- Inject API-level faults targeting specific AWS service calls (STS, DynamoDB, Security Hub)
- Understand how SQS visibility timeout interacts with slow/failed processing
- Validate idempotency and DLQ behavior under real failure conditions

**Prerequisites:**
- Existing ECS Fargate cluster with the Java processor task running
- SQS FIFO queue with DLQ configured
- Cross-account IAM roles for DynamoDB access
- FIS experiment IAM role (Module 2)

---

## Architecture Under Test

```
                         Account A                      │        Account B
                                                        │
 ┌──────────────┐    ┌─────────────────────────┐        │  ┌─────────────────┐
 │ SQS FIFO     │───>│  ECS Fargate Task (Java) │        │  │ DynamoDB Tables  │
 │              │    │                          │        │  │                  │
 │ maxReceive=3 │    │  1. ReceiveMessage       │        │  │  source-table    │
 │              │    │  2. AssumeRole (Acct B)  │───STS──│─>│  (read)          │
 │ ┌──────────┐ │    │  3. Read source-table    │───DDB──│─>│                  │
 │ │ DLQ      │ │    │  4. BatchImportFindings  │───>SecHub  │  results-table   │
 │ │ (FIFO)   │ │    │  5. Write results-table  │───DDB──│─>│  (write)         │
 │ └──────────┘ │    │  6. DeleteMessage        │        │  │                  │
 │              │<───│                          │        │  │                  │
 └──────────────┘    └─────────────────────────┘        │  └─────────────────┘
                              │
                     visibilityTimeout = 300s
                     maxReceiveCount = 3
```

**Key design details to keep in mind:**
- SQS FIFO provides exactly-once processing semantics *only if* the consumer correctly deletes the message
- `visibilityTimeout` (assume 300s / 5 min) determines how long before a failed message becomes visible again
- `maxReceiveCount: 3` means after 3 failed attempts, the message goes to DLQ
- Cross-account DDB access uses `sts:AssumeRole` to get temporary credentials for Account B
- `BatchImportFindings` can partially succeed (some findings imported, some rejected)

---

## IAM: Experiment Role Additions

Add these permissions to your FIS experiment role for API fault injection:

```json
{
  "Sid": "FISApiInjectionActions",
  "Effect": "Allow",
  "Action": [
    "fis:InjectApiInternalError",
    "fis:InjectApiThrottleError",
    "fis:InjectApiUnavailableError"
  ],
  "Resource": "arn:aws:fis:*:*:experiment/*"
}
```

The API fault injection actions target an **IAM role**, not the resource itself. You need to know the **task execution role** or **task role** ARN that your ECS task uses to make AWS API calls.

```bash
# Find your ECS task role ARN
TASK_ROLE_ARN=$(aws ecs describe-task-definition \
  --task-definition YOUR_TASK_DEF_NAME \
  --query 'taskDefinition.taskRoleArn' --output text)

echo "Task role: $TASK_ROLE_ARN"
```

---

## Scenario 1: Task Dies Mid-Processing

**The real-world failure:** ECS kills the task (OOM, spot reclamation, deployment, platform issue) while it's between reading DDB and deleting the SQS message.

**What can go wrong:**
- Message becomes visible again after `visibilityTimeout` → reprocessed
- If processing is NOT idempotent: duplicate Security Hub findings, duplicate DDB writes
- If the task wrote to results-table but didn't delete the SQS message: partial state

**Hypothesis:** "When a task is killed mid-processing, the message returns to the queue, gets reprocessed by another task, and no duplicate findings are created in Security Hub."

### Experiment Template

Save as `scenario1-task-kill.json`:
```json
{
  "description": "ECS SQS Processor: Kill task mid-processing",
  "targets": {
    "ecs-tasks": {
      "resourceType": "aws:ecs:task",
      "resourceTags": {
        "Application": ["sqs-processor"]
      },
      "filters": [
        {
          "path": "ClusterArn",
          "values": ["arn:aws:ecs:us-east-1:ACCOUNT_A:cluster/YOUR_CLUSTER"]
        }
      ],
      "selectionMode": "COUNT(1)"
    }
  },
  "actions": {
    "KillTask": {
      "actionId": "aws:ecs:stop-task",
      "targets": {
        "Tasks": "ecs-tasks"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::ACCOUNT_A:role/FISExperimentRole",
  "tags": {
    "Scenario": "1-task-kill"
  }
}
```

### How to Run

```bash
# Step 1: Send a test message to the queue BEFORE starting the experiment
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT_A/your-queue.fifo \
  --message-body '{"test": "scenario-1", "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' \
  --message-group-id "test" \
  --message-deduplication-id "scenario1-$(date +%s)"

# Step 2: Wait for the task to pick up the message (check logs)
# Give it ~10 seconds to start processing

# Step 3: Kill the task
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://scenario1-task-kill.json \
  --query 'experimentTemplate.id' --output text)

aws fis start-experiment --experiment-template-id "$TEMPLATE_ID"
```

### What to Observe

```bash
# 1. Did ECS launch a replacement task?
watch -n 5 "aws ecs list-tasks \
  --cluster YOUR_CLUSTER \
  --service-name YOUR_SERVICE \
  --query 'taskArns' --output text"

# 2. Check SQS - is the message back in the queue?
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT_A/your-queue.fifo \
  --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible

# 3. After reprocessing: check for duplicates in Security Hub
aws securityhub get-findings \
  --filters '{"Title":[{"Value":"scenario-1","Comparison":"CONTAINS"}]}' \
  --query 'Findings | length(@)'

# 4. Check DLQ after 3 failures (if applicable)
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT_A/your-queue-dlq.fifo \
  --attribute-names ApproximateNumberOfMessages
```

### What to Look For

| Check | Good | Bad |
|-------|------|-----|
| Replacement task launched | Within 30-60s | Task not replaced, service stuck at desired-1 |
| Message reprocessed | Shows up in SQS, picked up by new task | Message lost (deleted before processing completed) |
| No duplicate findings | Same finding count as expected | Double the findings → idempotency broken |
| No duplicate DDB writes | Single row in results-table | Duplicate or conflicting rows |

> **Pro Tip**: If you find duplicate writes, your app needs a deduplication key. For FIFO SQS, the `MessageId` or `MessageDeduplicationId` is a natural idempotency key. Write it to DDB as a condition expression: `attribute_not_exists(messageId)`.

---

## Scenario 2: Cross-Account AssumeRole Failure

**The real-world failure:** STS is throttled or returns transient errors. Your app can't get temporary credentials for Account B. This happens during credential rotation, under high concurrency, or during regional service degradation.

**What can go wrong:**
- App logs `ExpiredTokenException` or `ThrottlingException` from STS
- If the app doesn't retry: the message fails, gets retried by SQS (good) or goes to DLQ
- If the app caches credentials and they expire during processing: mid-flight failure

**Hypothesis:** "When STS AssumeRole is throttled at 50%, processing throughput degrades but no messages are lost. Failed messages return to the queue and eventually succeed."

### Experiment Template

Save as `scenario2-sts-throttle.json`:
```json
{
  "description": "ECS SQS Processor: STS AssumeRole throttling",
  "targets": {
    "task-role": {
      "resourceType": "aws:iam:role",
      "resourceArns": [
        "TASK_ROLE_ARN"
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "ThrottleSTS": {
      "actionId": "aws:fis:inject-api-throttle-error",
      "parameters": {
        "duration": "PT3M",
        "service": "sts",
        "operations": "AssumeRole",
        "percentage": "50"
      },
      "targets": {
        "Roles": "task-role"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::ACCOUNT_A:role/FISExperimentRole",
  "tags": {
    "Scenario": "2-sts-throttle"
  }
}
```

> **Important**: The target is the ECS **task role** (not the FIS experiment role). FIS injects faults into API calls made by the target role.

### What to Observe

```bash
# 1. Watch ECS task logs for STS errors
aws logs tail /ecs/your-task-log-group --follow --format short

# 2. Watch SQS message count (should stay stable or slowly drain)
watch -n 10 "aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT_A/your-queue.fifo \
  --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible \
  --output table"

# 3. Check DLQ (messages should NOT end up here from transient STS errors)
watch -n 10 "aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT_A/your-queue-dlq.fifo \
  --attribute-names ApproximateNumberOfMessages --output text"
```

### What to Look For

| Check | Good | Bad |
|-------|------|-----|
| App logs show retries | `Retrying AssumeRole (attempt 2/3)` | `AssumeRole failed, skipping message` |
| Messages not lost | SQS message count stable, messages reprocessed | Messages in DLQ after 3 receive attempts |
| Recovery after experiment | Processing resumes at normal rate | Task stuck, needs restart |
| Credential caching | App refreshes credentials on `ExpiredTokenException` | App crashes or uses stale credentials indefinitely |

> **Pro Tip**: If your Java app uses the AWS SDK v2 `StsAssumeRoleCredentialsProvider`, it has built-in caching and refresh. Verify it's configured with a reasonable `asyncCredentialUpdateEnabled(true)` to pre-refresh credentials before expiry.

---

## Scenario 3: DynamoDB Throttling

**The real-world failure:** DynamoDB returns `ProvisionedThroughputExceededException` on reads or writes. Common causes: hot partition, burst capacity exhausted, on-demand table hitting account limits.

**What can go wrong:**
- Processing slows down → SQS visibility timeout expires → duplicate processing
- App retries exhaust all attempts → message goes to DLQ
- Write to results-table fails but read from source-table succeeded → partial state

**Hypothesis:** "When DynamoDB is throttled at 70%, processing slows but eventually completes. Messages that can't be processed within 3 attempts land in the DLQ with enough info to replay."

### Experiment Template

Save as `scenario3-ddb-throttle.json`:
```json
{
  "description": "ECS SQS Processor: DynamoDB throttling",
  "targets": {
    "task-role": {
      "resourceType": "aws:iam:role",
      "resourceArns": [
        "TASK_ROLE_ARN"
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "ThrottleDDB": {
      "actionId": "aws:fis:inject-api-throttle-error",
      "parameters": {
        "duration": "PT5M",
        "service": "dynamodb",
        "operations": "GetItem,PutItem,Query",
        "percentage": "70"
      },
      "targets": {
        "Roles": "task-role"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::ACCOUNT_A:role/FISExperimentRole",
  "tags": {
    "Scenario": "3-ddb-throttle"
  }
}
```

### What to Observe

```bash
# 1. Watch processing latency in app logs
aws logs tail /ecs/your-task-log-group --follow \
  --filter-pattern "ThrottlingException"

# 2. Watch SQS - are messages piling up?
watch -n 10 "aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT_A/your-queue.fifo \
  --attribute-names All --output json | jq '{
    InFlight: .Attributes.ApproximateNumberOfMessagesNotVisible,
    Visible: .Attributes.ApproximateNumberOfMessages,
    Delayed: .Attributes.ApproximateNumberOfMessagesDelayed
  }'"

# 3. After experiment ends: did the backlog drain?
# (wait a few minutes and check again)
```

### The Visibility Timeout Trap

This is the most important thing to validate:

```
Timeline of a single message:

T+0s:    Task receives message (visibility timeout starts: 300s)
T+10s:   Read from source-table → THROTTLED, retry
T+30s:   Read succeeds
T+40s:   BatchImportFindings → OK
T+50s:   Write to results-table → THROTTLED, retry
T+70s:   Write → THROTTLED, retry
T+110s:  Write succeeds
T+111s:  DeleteMessage → OK
         Total: 111s ← well within 300s visibility timeout ✓

But at 70% throttle rate with SDK backoff:

T+0s:    Receive message
T+10s:   Read → THROTTLED
T+20s:   Read → THROTTLED (backoff: 2s)
T+32s:   Read → THROTTLED (backoff: 4s)
T+46s:   Read → OK
T+56s:   SecHub → OK
T+66s:   Write → THROTTLED
T+78s:   Write → THROTTLED (backoff: 2s)
...
T+290s:  Write → THROTTLED
T+300s:  ⚠️ VISIBILITY TIMEOUT EXPIRES - message visible again!
T+301s:  Another task receives the same message → DUPLICATE PROCESSING
```

**If you see this happening**, your options are:
1. Increase `visibilityTimeout` (but this increases retry delay for genuine failures)
2. Extend visibility timeout during processing: call `ChangeMessageVisibility` midway
3. Reduce SDK retry attempts and let the message return to queue faster

---

## Scenario 4: Security Hub API Throttling

**The real-world failure:** `BatchImportFindings` has a rate limit of 10 TPS per account per region. Under load or with multiple producers, you hit it.

**What can go wrong:**
- `TooManyRequestsException` from Security Hub
- Partial success: `BatchImportFindings` returns `FailedCount > 0` with `FailedFindings` array
- App may not handle partial failures → some findings silently dropped

**Hypothesis:** "When Security Hub is throttled at 50%, the app retries failed findings and no findings are lost."

### Experiment Template

Save as `scenario4-sechub-throttle.json`:
```json
{
  "description": "ECS SQS Processor: Security Hub API throttling",
  "targets": {
    "task-role": {
      "resourceType": "aws:iam:role",
      "resourceArns": [
        "TASK_ROLE_ARN"
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "ThrottleSecHub": {
      "actionId": "aws:fis:inject-api-throttle-error",
      "parameters": {
        "duration": "PT3M",
        "service": "securityhub",
        "operations": "BatchImportFindings",
        "percentage": "50"
      },
      "targets": {
        "Roles": "task-role"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::ACCOUNT_A:role/FISExperimentRole",
  "tags": {
    "Scenario": "4-sechub-throttle"
  }
}
```

### What to Look For

| Check | Good | Bad |
|-------|------|-----|
| App retries on `TooManyRequestsException` | Yes, with backoff | Throws exception, message goes to DLQ |
| Partial failures handled | App reads `FailedFindings`, retries only those | App ignores `FailedCount`, deletes message anyway |
| Finding count matches expected | All findings eventually imported | Missing findings (silent data loss) |

> **Pro Tip**: `BatchImportFindings` returns `FailedCount` and `SuccessCount` in the response. Many apps only check the HTTP status code (200) and miss the partial failures. This experiment will expose that.

```java
// BAD: Ignores partial failures
securityHubClient.batchImportFindings(request);
// assumes success if no exception

// GOOD: Handles partial failures
BatchImportFindingsResponse response = securityHubClient.batchImportFindings(request);
if (response.failedCount() > 0) {
    List<ImportFindingsError> failures = response.failedFindings();
    // retry only the failed findings
}
```

---

## Scenario 5: Slow Processing → Visibility Timeout Expiry

**The real-world failure:** Network latency or CPU pressure makes every API call slower. Total processing time exceeds SQS visibility timeout. The message becomes visible while still being processed → duplicate processing.

**This is the subtle, hard-to-catch-in-testing failure that chaos engineering is made for.**

**Hypothesis:** "When network latency adds 500ms to every call, processing stays within the visibility timeout and no duplicate processing occurs."

### Experiment Template

Save as `scenario5-slow-processing.json`:
```json
{
  "description": "ECS SQS Processor: Network latency causing slow processing",
  "targets": {
    "ecs-tasks": {
      "resourceType": "aws:ecs:task",
      "resourceTags": {
        "Application": ["sqs-processor"]
      },
      "filters": [
        {
          "path": "ClusterArn",
          "values": ["arn:aws:ecs:us-east-1:ACCOUNT_A:cluster/YOUR_CLUSTER"]
        }
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "InjectLatency": {
      "actionId": "aws:ecs:task-network-latency",
      "parameters": {
        "duration": "PT5M",
        "delayMilliseconds": "500",
        "jitterMilliseconds": "200",
        "sources": "0.0.0.0/0"
      },
      "targets": {
        "Tasks": "ecs-tasks"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::ACCOUNT_A:role/FISExperimentRole",
  "tags": {
    "Scenario": "5-slow-processing"
  }
}
```

### Escalation Variant: CPU Stress

If network latency alone doesn't push you past the visibility timeout, combine it with CPU stress:

Save as `scenario5b-combined-slow.json`:
```json
{
  "description": "ECS SQS Processor: Combined latency + CPU stress",
  "targets": {
    "ecs-tasks": {
      "resourceType": "aws:ecs:task",
      "resourceTags": {
        "Application": ["sqs-processor"]
      },
      "filters": [
        {
          "path": "ClusterArn",
          "values": ["arn:aws:ecs:us-east-1:ACCOUNT_A:cluster/YOUR_CLUSTER"]
        }
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "InjectLatency": {
      "actionId": "aws:ecs:task-network-latency",
      "parameters": {
        "duration": "PT5M",
        "delayMilliseconds": "500",
        "jitterMilliseconds": "200",
        "sources": "0.0.0.0/0"
      },
      "targets": {
        "Tasks": "ecs-tasks"
      }
    },
    "StressCPU": {
      "actionId": "aws:ecs:task-cpu-stress",
      "parameters": {
        "duration": "PT5M",
        "percent": "70"
      },
      "targets": {
        "Tasks": "ecs-tasks"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::ACCOUNT_A:role/FISExperimentRole",
  "tags": {
    "Scenario": "5b-combined-slow"
  }
}
```

### What to Observe

```bash
# 1. Monitor SQS receive count per message
#    If a message is received more than once, you have a visibility timeout problem
aws logs tail /ecs/your-task-log-group --follow \
  --filter-pattern "ReceiveMessage"

# 2. Watch for the telltale sign: ApproximateReceiveCount > 1
#    This appears in the SQS message attributes when your app reads it
#    (your app should log this value)

# 3. Check SQS metrics in CloudWatch
#    NumberOfMessagesReceived vs NumberOfMessagesDeleted
#    If received >> deleted, you have duplicate processing
```

### Key Math

```
Your visibility timeout should be:

  visibilityTimeout > (worst_case_processing_time * safety_margin)

With 500ms latency added per call and ~6 API calls per message:
  Normal:  6 calls × ~50ms  = ~300ms
  Latency: 6 calls × ~550ms = ~3.3s (+ SDK retries on throttles)

  With retries: 6 calls × 3 retries × 550ms = ~10s
  With backoff: could reach 30-60s per message

300s visibility timeout should be fine for this level of latency.

But what if latency is 2000ms AND DDB is throttled?
  6 calls × 5 retries × 2500ms (with backoff) = ~75s per attempt
  With multiple retry cycles: could reach 200-300s

  Now you're in the danger zone. ⚠️
```

---

## Running Order & Recommended Progression

| Order | Scenario | Duration | Why this order |
|-------|----------|----------|---------------|
| 1st | Task kill (Scenario 1) | 1 min | Simplest, validates basic SQS retry behavior |
| 2nd | Security Hub throttle (Scenario 4) | 3 min | Tests a common app-level bug (partial failures) |
| 3rd | STS throttle (Scenario 2) | 3 min | Tests credential handling under pressure |
| 4th | DDB throttle (Scenario 3) | 5 min | Tests retry + visibility timeout interaction |
| 5th | Slow processing (Scenario 5) | 5 min | The subtle killer -- run this last when you know the baseline |

**Between each scenario:** Send 5-10 test messages, let them process normally to confirm baseline behavior, then start the experiment.

---

## Combined Monitoring Dashboard

Create this before running experiments:

```bash
aws cloudwatch put-dashboard \
  --dashboard-name ECS-SQS-Processor-FIS \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "title": "SQS - Messages In Flight vs Visible",
          "metrics": [
            ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "your-queue.fifo"],
            ["AWS/SQS", "ApproximateNumberOfMessagesNotVisible", "QueueName", "your-queue.fifo"]
          ],
          "period": 60,
          "stat": "Average"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "SQS - DLQ Depth",
          "metrics": [
            ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "your-queue-dlq.fifo"]
          ],
          "period": 60,
          "stat": "Maximum"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "ECS - Task Count",
          "metrics": [
            ["AWS/ECS", "RunningTaskCount", "ServiceName", "YOUR_SERVICE", "ClusterName", "YOUR_CLUSTER"]
          ],
          "period": 60,
          "stat": "Average"
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "ECS - CPU Utilization",
          "metrics": [
            ["AWS/ECS", "CPUUtilization", "ServiceName", "YOUR_SERVICE", "ClusterName", "YOUR_CLUSTER"]
          ],
          "period": 60,
          "stat": "Average"
        }
      }
    ]
  }'
```

---

## Common Bugs This Lab Exposes

| Bug | Scenario | Fix |
|-----|----------|-----|
| Duplicate Security Hub findings | 1, 5 | Use message deduplication ID as idempotency key |
| Silent data loss from partial `BatchImportFindings` | 4 | Check `FailedCount` in response, retry failed items |
| Messages going to DLQ on transient errors | 2, 3 | Increase `maxReceiveCount`, add exponential backoff in app |
| Stale cross-account credentials | 2 | Use `StsAssumeRoleCredentialsProvider` with async refresh |
| Visibility timeout expiry | 3, 5 | Call `ChangeMessageVisibility` during long processing, or increase timeout |
| App doesn't log `ApproximateReceiveCount` | 1, 5 | Log it on every receive -- it's your duplicate processing canary |
| No circuit breaker on Security Hub calls | 4 | Add circuit breaker pattern; when tripped, leave message for retry |

---

## Cleanup

```bash
# Delete experiment templates
for f in scenario1-task-kill scenario2-sts-throttle scenario3-ddb-throttle \
         scenario4-sechub-throttle scenario5-slow-processing scenario5b-combined-slow; do
  TEMPLATE_ID=$(aws fis list-experiment-templates \
    --query "experimentTemplates[?tags.Scenario=='${f#scenario}'].id | [0]" \
    --output text 2>/dev/null)
  if [ "$TEMPLATE_ID" != "None" ] && [ -n "$TEMPLATE_ID" ]; then
    aws fis delete-experiment-template --id "$TEMPLATE_ID"
    echo "Deleted template for $f"
  fi
done

# Delete CloudWatch dashboard
aws cloudwatch delete-dashboards --dashboard-names ECS-SQS-Processor-FIS

# Drain any remaining test messages from DLQ
echo "Check DLQ for remaining messages:"
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT_A/your-queue-dlq.fifo \
  --attribute-names ApproximateNumberOfMessages --output text
```

---

## Knowledge Checkpoint

1. Why does FIS API fault injection target an **IAM role** rather than the resource itself?
2. A message has `ApproximateReceiveCount: 4` but `maxReceiveCount` is 3. How is this possible?
3. `BatchImportFindings` returns HTTP 200 with `FailedCount: 2`. Is this a success or failure?
4. Your SQS visibility timeout is 300s and processing normally takes 10s. Under what condition from this lab could the timeout expire?
5. What's the difference between the FIS action `aws:ecs:stop-task` and a Kubernetes pod delete?

<details>
<summary>Answers</summary>

1. FIS intercepts AWS API calls made by the target IAM role at the AWS API layer. It doesn't need access to the calling application -- it operates at the SDK/API level. This means any process using that role gets affected.

2. The message was received 3 times and failed, going to DLQ. It was then redrive from DLQ (or another consumer read from the DLQ), incrementing the count past `maxReceiveCount`. The count is cumulative across the lifetime of the message.

3. It's a partial success. HTTP 200 means the API call itself succeeded, but 2 findings were rejected. Your app MUST check `FailedCount` and handle the `FailedFindings` array. This is one of the most common silent data loss bugs.

4. When DDB throttling (Scenario 3) causes multiple retries with exponential backoff, or when network latency (Scenario 5) is combined with CPU stress, total processing time can approach or exceed 300s. Especially if the SDK retry config allows many retries with long backoff.

5. Both kill the workload, but ECS `stop-task` works at the AWS control plane level (it calls the ECS API to stop the task). Kubernetes pod delete works at the kubelet level. For Fargate tasks, `stop-task` is the correct approach since there's no Kubernetes layer.

</details>
