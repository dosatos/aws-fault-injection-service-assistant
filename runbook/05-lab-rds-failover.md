# Module 5: Lab 3 - RDS Failover Testing

**Time: 2 hours**

**Learning Objectives:**
- Trigger an Aurora/RDS Multi-AZ failover with FIS
- Measure failover time and application impact
- Understand the difference between cluster failover and instance reboot
- Validate your application's database reconnection logic

**Prerequisites:** Module 3 completed, familiarity with RDS/Aurora basics

---

## Objective

Force a database failover and measure how long your application is unable to write to the database. This is one of the most valuable FIS experiments for production systems.

**Hypothesis:** "When the primary Aurora writer instance fails over, our application reconnects within 30 seconds and no transactions are lost."

---

## 5.1 Setup: Aurora Cluster for Testing

> If you already have an RDS Multi-AZ or Aurora cluster, skip to 5.2 and adjust tags.

Add this to your lab environment or deploy separately (`lab5-rds.yaml`):

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: FIS Lab - Aurora cluster for failover testing

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC from Module 2 lab environment
  SubnetAId:
    Type: AWS::EC2::Subnet::Id
  SubnetBId:
    Type: AWS::EC2::Subnet::Id

Resources:
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: FIS Lab DB Subnets
      SubnetIds:
        - !Ref SubnetAId
        - !Ref SubnetBId

  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: FIS Lab RDS SG
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          CidrIp: 10.0.0.0/16

  AuroraCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      DBClusterIdentifier: fis-lab-aurora
      Engine: aurora-mysql
      EngineVersion: '8.0.mysql_aurora.3.04.0'
      MasterUsername: admin
      ManageMasterUserPassword: true
      DBSubnetGroupName: !Ref DBSubnetGroup
      VpcSecurityGroupIds:
        - !Ref DBSecurityGroup
      Tags:
        - Key: Environment
          Value: fis-lab

  AuroraWriter:
    Type: AWS::RDS::DBInstance
    Properties:
      DBClusterIdentifier: !Ref AuroraCluster
      DBInstanceClass: db.t3.medium
      Engine: aurora-mysql
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Environment
          Value: fis-lab
        - Key: Role
          Value: writer

  AuroraReader:
    Type: AWS::RDS::DBInstance
    Properties:
      DBClusterIdentifier: !Ref AuroraCluster
      DBInstanceClass: db.t3.medium
      Engine: aurora-mysql
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Environment
          Value: fis-lab
        - Key: Role
          Value: reader

Outputs:
  ClusterEndpoint:
    Value: !GetAtt AuroraCluster.Endpoint.Address
  ReaderEndpoint:
    Value: !GetAtt AuroraCluster.ReadEndpoint.Address
  ClusterIdentifier:
    Value: !Ref AuroraCluster
```

```bash
aws cloudformation create-stack \
  --stack-name fis-lab-rds \
  --template-body file://lab5-rds.yaml \
  --parameters \
    ParameterKey=VpcId,ParameterValue=vpc-YOUR_VPC_ID \
    ParameterKey=SubnetAId,ParameterValue=subnet-YOUR_SUBNET_A \
    ParameterKey=SubnetBId,ParameterValue=subnet-YOUR_SUBNET_B

aws cloudformation wait stack-create-complete --stack-name fis-lab-rds
```

> **Cost Warning**: Aurora db.t3.medium instances cost ~$0.07/hr each. That's ~$3.36/day for both instances. Tear down promptly.

---

## 5.2 Experiment: Aurora Cluster Failover

### FIS Actions for RDS

| Action | What It Does | When to Use |
|--------|-------------|-------------|
| `aws:rds:failover-db-cluster` | Promotes a reader to writer, demotes the current writer | Test failover resilience |
| `aws:rds:reboot-db-instances` | Reboots the DB instance (with optional force-failover) | Test instance restart handling |

### CLI: Create and Run

Save as `lab5-template.json`:
```json
{
  "description": "Lab 5: Aurora cluster failover",
  "targets": {
    "aurora-cluster": {
      "resourceType": "aws:rds:cluster",
      "resourceArns": [
        "arn:aws:rds:us-east-1:YOUR_ACCOUNT_ID:cluster:fis-lab-aurora"
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "FailoverAurora": {
      "actionId": "aws:rds:failover-db-cluster",
      "targets": {
        "Clusters": "aurora-cluster"
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
    "Lab": "5",
    "Environment": "fis-lab"
  }
}
```

```bash
TEMPLATE_ID=$(aws fis create-experiment-template \
  --cli-input-json file://lab5-template.json \
  --query 'experimentTemplate.id' --output text)

# Record the current writer
echo "Before failover:"
aws rds describe-db-clusters \
  --db-cluster-identifier fis-lab-aurora \
  --query 'DBClusters[0].DBClusterMembers[].{Instance:DBInstanceIdentifier,IsWriter:IsClusterWriter}' \
  --output table

# Start the experiment
EXPERIMENT_ID=$(aws fis start-experiment \
  --experiment-template-id "$TEMPLATE_ID" \
  --query 'experiment.id' --output text)

echo "Experiment started: $EXPERIMENT_ID"
```

### Monitoring Failover

```bash
# Monitor cluster members -- watch the writer swap
watch -n 5 "aws rds describe-db-clusters \
  --db-cluster-identifier fis-lab-aurora \
  --query 'DBClusters[0].{Status:Status,Members:DBClusterMembers[].{Instance:DBInstanceIdentifier,IsWriter:IsClusterWriter}}' \
  --output json"

# Check RDS events for failover timing
aws rds describe-events \
  --source-identifier fis-lab-aurora \
  --source-type db-cluster \
  --duration 30 \
  --query 'Events[].{Time:Date,Message:Message}' \
  --output table
```

**Expected output during failover:**
```
Before:                          After:
Instance         IsWriter        Instance         IsWriter
fis-lab-aurora-1  true           fis-lab-aurora-1  false
fis-lab-aurora-2  false          fis-lab-aurora-2  true    <-- promoted
```

**Typical Aurora failover time**: 15-35 seconds.

### Measuring Application Impact

If you have an application connected to the cluster endpoint, time the disruption:

```bash
# Simple MySQL connectivity test (requires mysql client)
# Run this BEFORE starting the experiment
while true; do
  START=$(date +%s%N)
  mysql -h fis-lab-aurora.cluster-xxxx.us-east-1.rds.amazonaws.com \
    -u admin -p'password' -e "SELECT 1" 2>&1 | head -1
  END=$(date +%s%N)
  ELAPSED=$(( (END - START) / 1000000 ))
  echo "$(date +%H:%M:%S) - ${ELAPSED}ms"
  sleep 1
done
```

---

## 5.3 Terraform

```hcl
resource "aws_fis_experiment_template" "lab5_rds_failover" {
  description = "Lab 5: Aurora cluster failover"
  role_arn    = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/FISExperimentRole"

  stop_condition {
    source = "none"
  }

  action {
    name      = "FailoverAurora"
    action_id = "aws:rds:failover-db-cluster"

    target {
      key   = "Clusters"
      value = "aurora-cluster"
    }
  }

  target {
    name           = "aurora-cluster"
    resource_type  = "aws:rds:cluster"
    selection_mode = "ALL"

    resource_arns = [
      "arn:aws:rds:us-east-1:${data.aws_caller_identity.current.account_id}:cluster:fis-lab-aurora"
    ]
  }

  tags = {
    Lab         = "5"
    Environment = "fis-lab"
  }
}
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `InvalidDBClusterStateFault` | Cluster is not in `available` state | Wait for cluster to be fully available before running |
| Failover doesn't happen | Single-instance cluster (no reader) | Ensure you have at least 2 instances in the cluster |
| Application can't reconnect | Using instance endpoint instead of cluster endpoint | Always use the cluster endpoint (`*.cluster-*.rds.amazonaws.com`) which follows the writer |
| Failover takes > 60s | Large uncommitted transactions | Check for long-running transactions before testing |

> **Pro Tip**: Always use the Aurora **cluster endpoint** for writes and the **reader endpoint** for reads. Instance endpoints don't follow failover.

---

## Cleanup

```bash
aws fis delete-experiment-template --id "$TEMPLATE_ID"

# Delete the RDS stack (saves ~$3.36/day)
aws cloudformation delete-stack --stack-name fis-lab-rds
aws cloudformation wait stack-delete-complete --stack-name fis-lab-rds
```

---

## What You Learned

- How to trigger Aurora failover via FIS
- The difference between `failover-db-cluster` and `reboot-db-instances`
- How to measure failover duration using RDS events
- Why cluster endpoints are critical for failover resilience
- Typical Aurora failover times (~15-35s)

---

## Knowledge Checkpoint

1. What happens to the reader instance during an Aurora cluster failover?
2. Why should applications use cluster endpoints instead of instance endpoints?
3. What's the typical Aurora failover time?
4. How would you test a non-Aurora RDS Multi-AZ failover with FIS?

<details>
<summary>Answers</summary>

1. The reader is promoted to writer, and the old writer becomes a reader (role swap).
2. Cluster endpoints automatically point to the current writer. Instance endpoints are static and won't follow failover.
3. Typically 15-35 seconds for Aurora.
4. Use `aws:rds:reboot-db-instances` with `forceFailover: true` parameter on a Multi-AZ RDS instance.

</details>
