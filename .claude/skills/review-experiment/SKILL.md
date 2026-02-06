---
name: review-experiment
description: Audit an AWS FIS experiment template (JSON or Terraform) for safety issues, best practice violations, and correctness. Provide it as a file path or paste it.
argument-hint: "[file path to experiment template, e.g. 'template.json']"
---

# Review FIS Experiment Template

You are auditing an AWS FIS experiment template for safety, correctness, and best practices.

## Input

If `$ARGUMENTS` is a file path, read that file. Otherwise, ask the user to paste their experiment template (JSON or HCL).

## Audit Checklist

Run through every check below. Report each as PASS, WARN, or FAIL with a one-line explanation.

### 1. Action Validity
- [ ] Every `actionId` exists in the official FIS actions list (@runbook/10-reference.md)
- [ ] Action parameters are valid for the action type (e.g., `duration` format is ISO 8601)
- [ ] Duration values are reasonable (not accidentally hours when minutes intended)

### 2. Target Safety
- [ ] Targets use `resourceTags` or `resourceArns` (not unscoped)
- [ ] `selectionMode` is appropriate for the environment
  - `ALL` in production → **FAIL** unless explicitly justified
  - `PERCENT(>25)` in production → **WARN**
  - `COUNT(1)` in production → **PASS** (recommended starting point)
- [ ] Filters are present where applicable (e.g., `State.Name: running` for EC2)
- [ ] If targeting EKS pods: `kubernetesServiceAccount` parameter is set

### 3. Stop Conditions
- [ ] At least one stop condition is present (non-`none`)
  - Exception: `"source": "none"` is acceptable only if template tags indicate `dev` or `lab`
- [ ] Stop condition alarm ARN is properly formed
- [ ] For production: recommend multiple stop conditions (error rate AND latency)

### 4. Reversibility
- [ ] `aws:ec2:stop-instances` has `startInstancesAfterDuration` set → **WARN** if missing (instance won't auto-restart)
- [ ] `aws:ec2:terminate-instances` present → **WARN** always (irreversible, confirm intentional)
- [ ] Network disruption actions have reasonable duration (≤ 10 min for first run)

### 5. IAM Role
- [ ] `roleArn` is present and not empty
- [ ] Role name suggests scoped permissions (not `AdministratorAccess` or overly broad names)
- [ ] If the template modifies cross-account resources, remind about AssumeRole setup

### 6. Sequencing
- [ ] If multiple actions exist, check for `startAfter` dependencies — are they logical?
- [ ] Parallel actions don't conflict (e.g., stop and terminate the same instance)
- [ ] `aws:fis:wait` actions have reasonable durations for observation

### 7. Tags and Metadata
- [ ] Template has `tags` with at least `Environment`
- [ ] `description` field is present and meaningful (not empty or generic)

### 8. Cost Estimate
Calculate and report:
- Count of actions
- Sum of durations (parallel actions = max duration, sequential = sum)
- Total action-minutes × $0.10
- If multi-account: multiply by account count

## Output Format

```
## FIS Experiment Review: [template description]

### Summary
- Actions: [count]
- Targets: [count]
- Estimated cost: $X.XX
- Blast radius: [description]
- Environment: [detected from tags or inferred]

### Findings

| # | Severity | Check | Finding |
|---|----------|-------|---------|
| 1 | FAIL     | ...   | ...     |
| 2 | WARN     | ...   | ...     |
| 3 | PASS     | ...   | ...     |

### Recommendations
[Prioritized list of fixes, with code snippets for FAIL items]

### Matching Runbook Lab
[Point to the relevant runbook module for the experiment type]
```
