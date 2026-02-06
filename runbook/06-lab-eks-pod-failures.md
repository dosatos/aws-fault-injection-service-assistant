# Module 6: Lab 4 - EKS Pod Failures

**Time: 3 hours**

**Learning Objectives:**
- Delete pods and observe Kubernetes self-healing
- Inject CPU/memory stress into pods
- Inject network latency and packet loss at the pod level
- Understand FIS EKS action requirements (Kubernetes RBAC + IAM)

**Prerequisites:** Module 3 completed, working EKS cluster (or willingness to create one)

---

## Objective

Test Kubernetes workload resilience by killing pods, stressing resources, and disrupting pod networking. Validate that deployments self-heal and services remain available.

**Hypothesis:** "When 50% of our application pods are deleted, Kubernetes reschedules them within 60 seconds and service availability is maintained."

---

## 6.1 EKS Prerequisites

FIS EKS actions require additional setup beyond the standard IAM role.

### Kubernetes RBAC Configuration

FIS needs a Kubernetes `ServiceAccount` and RBAC bindings to manage pods:

```yaml
# fis-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fis-experiment
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fis-experiment-role
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "delete"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/ephemeralcontainers"]
    verbs: ["update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fis-experiment-binding
subjects:
  - kind: ServiceAccount
    name: fis-experiment
    namespace: default
roleRef:
  kind: ClusterRole
  name: fis-experiment-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f fis-rbac.yaml
```

### Update aws-auth ConfigMap

Map the FIS experiment IAM role to the Kubernetes service account:

```bash
# Add to aws-auth ConfigMap
kubectl edit configmap aws-auth -n kube-system
```

Add under `mapRoles`:
```yaml
- rolearn: arn:aws:iam::YOUR_ACCOUNT_ID:role/FISExperimentRole
  username: fis-experiment
  groups:
    - fis-experiment-role
```

### Deploy a Test Workload

```yaml
# test-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fis-test-app
  labels:
    app: fis-test
    environment: fis-lab
spec:
  replicas: 4
  selector:
    matchLabels:
      app: fis-test
  template:
    metadata:
      labels:
        app: fis-test
        environment: fis-lab
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: fis-test-svc
spec:
  selector:
    app: fis-test
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f test-app.yaml
kubectl get pods -l app=fis-test
```

---

## 6.2 Experiment: Pod Deletion

### CLI

Save as `lab6a-pod-delete.json`:
```json
{
  "description": "Lab 6A: Delete 50% of test pods",
  "targets": {
    "eks-pods": {
      "resourceType": "aws:eks:pod",
      "parameters": {
        "clusterIdentifier": "arn:aws:eks:us-east-1:YOUR_ACCOUNT_ID:cluster/YOUR_CLUSTER_NAME",
        "namespace": "default",
        "selectorType": "labelSelector",
        "selectorValue": "app=fis-test"
      },
      "selectionMode": "PERCENT(50)"
    }
  },
  "actions": {
    "DeletePods": {
      "actionId": "aws:eks:pod-delete",
      "parameters": {
        "kubernetesServiceAccount": "fis-experiment"
      },
      "targets": {
        "Pods": "eks-pods"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/FISExperimentRole",
  "tags": {
    "Lab": "6a",
    "Environment": "fis-lab"
  }
}
```

```bash
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://lab6a-pod-delete.json \
  --query 'experimentTemplate.id' --output text)

# Watch pods in one terminal
kubectl get pods -l app=fis-test -w

# Start experiment in another terminal
aws fis start-experiment --experiment-template-id "$TEMPLATE_ID"
```

**Expected behavior:**
```
NAME                           READY   STATUS        AGE
fis-test-app-abc123-x1234      1/1     Terminating   5m    <-- killed by FIS
fis-test-app-abc123-y5678      1/1     Terminating   5m    <-- killed by FIS
fis-test-app-abc123-z9012      1/1     Running       5m
fis-test-app-abc123-w3456      1/1     Running       5m
fis-test-app-abc123-new01      0/1     Pending       1s    <-- Kubernetes rescheduling
fis-test-app-abc123-new02      0/1     Pending       1s    <-- Kubernetes rescheduling
```

Kubernetes should reschedule the deleted pods within seconds (for simple workloads).

---

## 6.3 Experiment: Pod CPU Stress

Save as `lab6b-cpu-stress.json`:
```json
{
  "description": "Lab 6B: CPU stress on test pods",
  "targets": {
    "eks-pods": {
      "resourceType": "aws:eks:pod",
      "parameters": {
        "clusterIdentifier": "arn:aws:eks:us-east-1:YOUR_ACCOUNT_ID:cluster/YOUR_CLUSTER_NAME",
        "namespace": "default",
        "selectorType": "labelSelector",
        "selectorValue": "app=fis-test"
      },
      "selectionMode": "COUNT(2)"
    }
  },
  "actions": {
    "StressCPU": {
      "actionId": "aws:eks:pod-cpu-stress",
      "parameters": {
        "duration": "PT3M",
        "percent": "80",
        "kubernetesServiceAccount": "fis-experiment"
      },
      "targets": {
        "Pods": "eks-pods"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/FISExperimentRole",
  "tags": {
    "Lab": "6b",
    "Environment": "fis-lab"
  }
}
```

```bash
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://lab6b-cpu-stress.json \
  --query 'experimentTemplate.id' --output text)

aws fis start-experiment --experiment-template-id "$TEMPLATE_ID"

# Monitor pod resource usage
kubectl top pods -l app=fis-test
```

---

## 6.4 Experiment: Pod Network Latency

Save as `lab6c-network-latency.json`:
```json
{
  "description": "Lab 6C: Inject 200ms latency on test pods",
  "targets": {
    "eks-pods": {
      "resourceType": "aws:eks:pod",
      "parameters": {
        "clusterIdentifier": "arn:aws:eks:us-east-1:YOUR_ACCOUNT_ID:cluster/YOUR_CLUSTER_NAME",
        "namespace": "default",
        "selectorType": "labelSelector",
        "selectorValue": "app=fis-test"
      },
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "InjectLatency": {
      "actionId": "aws:eks:pod-network-latency",
      "parameters": {
        "duration": "PT3M",
        "delayMilliseconds": "200",
        "jitterMilliseconds": "50",
        "kubernetesServiceAccount": "fis-experiment",
        "sources": "0.0.0.0/0"
      },
      "targets": {
        "Pods": "eks-pods"
      }
    }
  },
  "stopConditions": [
    {
      "source": "none"
    }
  ],
  "roleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/FISExperimentRole",
  "tags": {
    "Lab": "6c",
    "Environment": "fis-lab"
  }
}
```

**Available EKS network actions:**
| Action | Parameters | Use Case |
|--------|-----------|----------|
| `aws:eks:pod-network-latency` | `delayMilliseconds`, `jitterMilliseconds` | Simulate slow network |
| `aws:eks:pod-network-packet-loss` | `lossPercent` | Simulate unreliable network |
| `aws:eks:pod-network-blackhole-port` | `protocol`, `port`, `trafficType` | Simulate service dependency failure |

---

## 6.5 Terraform

```hcl
resource "aws_fis_experiment_template" "lab6_pod_delete" {
  description = "Lab 6: Delete 50% of test pods"
  role_arn    = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/FISExperimentRole"

  stop_condition {
    source = "none"
  }

  action {
    name      = "DeletePods"
    action_id = "aws:eks:pod-delete"

    parameter {
      key   = "kubernetesServiceAccount"
      value = "fis-experiment"
    }

    target {
      key   = "Pods"
      value = "eks-pods"
    }
  }

  target {
    name           = "eks-pods"
    resource_type  = "aws:eks:pod"
    selection_mode = "PERCENT(50)"

    parameters = {
      clusterIdentifier = "arn:aws:eks:us-east-1:${data.aws_caller_identity.current.account_id}:cluster/YOUR_CLUSTER_NAME"
      namespace         = "default"
      selectorType      = "labelSelector"
      selectorValue     = "app=fis-test"
    }
  }

  tags = {
    Lab         = "6"
    Environment = "fis-lab"
  }
}
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `UnauthorizedAccess` on EKS actions | FIS role not mapped in aws-auth | Add FIS role to aws-auth ConfigMap |
| Pod stress action fails | Missing RBAC for ephemeral containers | Add `pods/ephemeralcontainers` verb to ClusterRole |
| Network latency not injected | Pod security policy blocking tc commands | Ensure pods run with sufficient capabilities |
| Pods not selected | Wrong label selector or namespace | Verify with `kubectl get pods -l app=fis-test -n default` |
| Actions timeout | Cluster API server unreachable from FIS | Ensure cluster endpoint has public access or VPC endpoint configured |

> **Pro Tip**: FIS injects CPU/memory/network stress via ephemeral containers using the `stress-ng` and `tc` tools. These are sidecar-like processes that run alongside your container, so they accurately stress the pod's cgroup.

---

## Cleanup

```bash
aws fis delete-experiment-template --id "$TEMPLATE_ID"
kubectl delete -f test-app.yaml
kubectl delete -f fis-rbac.yaml
```

---

## What You Learned

- FIS EKS actions require both IAM roles AND Kubernetes RBAC
- Pod deletion tests Kubernetes self-healing (Deployment controller)
- CPU/memory stress tests resource limits and HPA behavior
- Network latency/loss tests service mesh and timeout configurations
- `PERCENT(n)` selection mode is ideal for gradual blast radius control

---

## Knowledge Checkpoint

1. Why do EKS FIS experiments require both IAM and Kubernetes RBAC configuration?
2. What mechanism does FIS use to inject CPU stress into pods?
3. What would happen if you delete 100% of pods in a Deployment with `replicas: 4`?
4. How would you test HPA (Horizontal Pod Autoscaler) response using FIS?

<details>
<summary>Answers</summary>

1. IAM is for FIS to access the EKS API. Kubernetes RBAC is for FIS to manage pods within the cluster. They're two separate authorization systems.
2. Ephemeral containers running `stress-ng`. FIS injects a sidecar process into the pod's cgroup.
3. All 4 pods would be deleted, but the Deployment controller would immediately schedule 4 new pods. Brief downtime until pods become Ready.
4. Use `aws:eks:pod-cpu-stress` with high CPU percentage on existing pods. HPA should detect increased CPU usage and scale out the deployment.

</details>
