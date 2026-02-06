# Module 2: Prerequisites & Setup

**Time: 3 hours**

**Learning Objectives:**
- Verify you have the required AWS knowledge
- Create a properly scoped IAM role for FIS experiments
- Deploy a sandbox lab environment
- Install and configure required tooling

---

## 2.1 Required AWS Knowledge Checklist

Before starting the labs, confirm you're comfortable with:

- [ ] **EC2**: Launch instances, security groups, key pairs
- [ ] **VPC**: Subnets, route tables, NACLs, internet gateways
- [ ] **IAM**: Roles, policies, trust relationships, AssumeRole
- [ ] **CloudWatch**: Metrics, alarms, dashboards
- [ ] **AWS CLI**: Basic usage (`aws configure`, running commands)
- [ ] **RDS** (for Module 5): Multi-AZ deployments, Aurora clusters
- [ ] **EKS** (for Module 6): Cluster basics, kubectl, pods/deployments

> If you're missing RDS or EKS knowledge, you can still do Modules 3-4 and come back.

---

## 2.2 IAM Setup

FIS requires two types of IAM configuration:

### A. Your User/Role Permissions (to manage FIS)

Attach this policy to your IAM user or role to create and run experiments:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FISManagement",
      "Effect": "Allow",
      "Action": [
        "fis:CreateExperimentTemplate",
        "fis:DeleteExperimentTemplate",
        "fis:GetExperimentTemplate",
        "fis:ListExperimentTemplates",
        "fis:UpdateExperimentTemplate",
        "fis:StartExperiment",
        "fis:StopExperiment",
        "fis:GetExperiment",
        "fis:ListExperiments",
        "fis:ListActions",
        "fis:GetAction",
        "fis:ListTargetResourceTypes",
        "fis:GetTargetResourceType",
        "fis:TagResource",
        "fis:UntagResource"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PassRoleToFIS",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::*:role/FISExperimentRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "fis.amazonaws.com"
        }
      }
    }
  ]
}
```

### B. FIS Experiment Role (what FIS can do to your resources)

This is the role FIS assumes when running experiments. **Scope it tightly.**

**Trust policy** (`fis-trust-policy.json`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "fis.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "YOUR_ACCOUNT_ID"
        }
      }
    }
  ]
}
```

**Permission policy** (`fis-experiment-policy.json`) -- this covers all labs in this runbook:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2Actions",
      "Effect": "Allow",
      "Action": [
        "ec2:StopInstances",
        "ec2:StartInstances",
        "ec2:RebootInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Environment": "fis-lab"
        }
      }
    },
    {
      "Sid": "NetworkActions",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkAcl",
        "ec2:CreateNetworkAclEntry",
        "ec2:DeleteNetworkAcl",
        "ec2:DeleteNetworkAclEntry",
        "ec2:DescribeNetworkAcls",
        "ec2:ReplaceNetworkAclAssociation",
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcs"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RDSActions",
      "Effect": "Allow",
      "Action": [
        "rds:FailoverDBCluster",
        "rds:RebootDBInstance",
        "rds:DescribeDBClusters",
        "rds:DescribeDBInstances"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EKSActions",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:DescribeNodegroup",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SSMActions",
      "Effect": "Allow",
      "Action": [
        "ssm:SendCommand",
        "ssm:GetCommandInvocation",
        "ssm:ListCommands",
        "ssm:CancelCommand"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchReadForStopConditions",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:DescribeAlarms"
      ],
      "Resource": "*"
    },
    {
      "Sid": "FISFaultInjectionActions",
      "Effect": "Allow",
      "Action": [
        "fis:InjectApiInternalError",
        "fis:InjectApiThrottleError",
        "fis:InjectApiUnavailableError"
      ],
      "Resource": "arn:aws:fis:*:*:experiment/*"
    }
  ]
}
```

**Create the role via CLI:**
```bash
# Replace YOUR_ACCOUNT_ID in fis-trust-policy.json first

aws iam create-role \
  --role-name FISExperimentRole \
  --assume-role-policy-document file://fis-trust-policy.json

aws iam put-role-policy \
  --role-name FISExperimentRole \
  --policy-name FISExperimentPolicy \
  --policy-document file://fis-experiment-policy.json

# Save the ARN -- you'll need it for every experiment
aws iam get-role --role-name FISExperimentRole \
  --query 'Role.Arn' --output text
```

> **Security Best Practice**: In production, scope the experiment role to specific resource ARNs and tags, not `"Resource": "*"`. The wildcard above is for lab convenience only.

---

## 2.3 Sandbox Lab Environment

Deploy a minimal sandbox with EC2 instances for the hands-on labs. Save this as `lab-environment.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: FIS Lab Sandbox Environment

Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64

Resources:
  # --- VPC ---
  LabVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: fis-lab-vpc
        - Key: Environment
          Value: fis-lab

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref LabVPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref LabVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: fis-lab-public-a
        - Key: Environment
          Value: fis-lab

  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref LabVPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: fis-lab-public-b
        - Key: Environment
          Value: fis-lab

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref LabVPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  SubnetARouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetA
      RouteTableId: !Ref PublicRouteTable

  SubnetBRouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetB
      RouteTableId: !Ref PublicRouteTable

  # --- Security Group ---
  LabSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: FIS Lab SG
      VpcId: !Ref LabVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Environment
          Value: fis-lab

  # --- EC2 Instances ---
  WebServerA:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId
      SubnetId: !Ref PublicSubnetA
      SecurityGroupIds:
        - !Ref LabSecurityGroup
      Tags:
        - Key: Name
          Value: fis-lab-web-a
        - Key: Environment
          Value: fis-lab
        - Key: Role
          Value: webserver

  WebServerB:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId
      SubnetId: !Ref PublicSubnetB
      SecurityGroupIds:
        - !Ref LabSecurityGroup
      Tags:
        - Key: Name
          Value: fis-lab-web-b
        - Key: Environment
          Value: fis-lab
        - Key: Role
          Value: webserver

  AppServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId
      SubnetId: !Ref PublicSubnetA
      SecurityGroupIds:
        - !Ref LabSecurityGroup
      Tags:
        - Key: Name
          Value: fis-lab-app
        - Key: Environment
          Value: fis-lab
        - Key: Role
          Value: appserver

Outputs:
  VpcId:
    Value: !Ref LabVPC
  SubnetAId:
    Value: !Ref PublicSubnetA
  SubnetBId:
    Value: !Ref PublicSubnetB
  WebServerAId:
    Value: !Ref WebServerA
  WebServerBId:
    Value: !Ref WebServerB
  AppServerId:
    Value: !Ref AppServer
```

**Deploy:**
```bash
aws cloudformation create-stack \
  --stack-name fis-lab-environment \
  --template-body file://lab-environment.yaml \
  --capabilities CAPABILITY_IAM

# Wait for completion
aws cloudformation wait stack-create-complete \
  --stack-name fis-lab-environment

# Get outputs
aws cloudformation describe-stacks \
  --stack-name fis-lab-environment \
  --query 'Stacks[0].Outputs' --output table
```

---

## 2.4 Tool Installation

```bash
# AWS CLI v2 (macOS)
brew install awscli
aws --version  # should be 2.x

# AWS CLI v2 (Linux)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Terraform
brew install terraform    # macOS
# or download from https://developer.hashicorp.com/terraform/install

# jq (for parsing JSON output)
brew install jq           # macOS
sudo apt-get install jq   # Debian/Ubuntu

# Verify
aws sts get-caller-identity
terraform version
jq --version
```

**Configure AWS CLI:**
```bash
aws configure
# Enter your Access Key, Secret Key, default region (e.g., us-east-1), output format (json)
```

---

## 2.5 Cleanup Checklist

Use this after every lab session to avoid runaway costs:

```bash
# Check for running experiments
aws fis list-experiments \
  --query 'experiments[?state.status==`running`]' --output table

# Stop any running experiments
aws fis stop-experiment --id EXPERIMENT_ID

# Delete the lab stack when done with all labs
aws cloudformation delete-stack --stack-name fis-lab-environment
aws cloudformation wait stack-delete-complete --stack-name fis-lab-environment

# Verify no lingering resources
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=fis-lab" \
  --query 'Reservations[].Instances[].{Id:InstanceId,State:State.Name}' \
  --output table
```

---

## Knowledge Checkpoint

1. Why does the FIS experiment role need a trust policy with `fis.amazonaws.com` as principal?
2. What tag are we using to scope our lab resources?
3. How would you restrict the experiment role to only stop (not terminate) EC2 instances?
4. What's the first command you should run if you suspect an experiment is still running?

<details>
<summary>Answers</summary>

1. Because FIS needs to assume this role via `sts:AssumeRole` to get permissions on your resources. The trust policy authorizes the FIS service to do this.
2. `Environment: fis-lab`
3. Remove `ec2:TerminateInstances` from the EC2Actions statement in the permission policy.
4. `aws fis list-experiments --query 'experiments[?state.status==`running`]'`

</details>
