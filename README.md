# CloudFormation Nested Stack Template : EC2 Instance

<!-- Row 1: Status - Most Important -->
[![Release](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)&nbsp;[![GitHub Repo](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/github/languages/top/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazon&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-ec2-instance/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/5f5b47db8b68709abe89209dcabdbeee/raw/cfn-nested-aws-ec2-instance.json)](https://gist.github.com/subhamay-bhattacharyya/5f5b47db8b68709abe89209dcabdbeee)

## Contents

- [EC2 Instance Template](#ec2-instance-template) — launch a single EC2 instance with optional instance profile, user data, and target group registration (`cloudformation/template.yaml`)
- [S3 Bucket Templates](#overview) — S3 bucket creation with security defaults and optional bucket policy enforcement (`templates/`)

## EC2 Instance Template

`cloudformation/template.yaml` is a nested CloudFormation template that creates a single EC2 instance. It can optionally create an instance profile for an existing IAM role, accept Base64-encoded user data, and register the instance with an existing target group.

### Features

- ✅ Latest Amazon Linux 2023 AMI by default (override with an AMI ID or SSM parameter)
- ✅ IMDSv2 required on the instance
- ✅ `Name` tag in the format `<ProjectName>-<InstanceBaseName>-<environment>-<AWS Region>`, with `-<CiSuffix>` appended when set
- ✅ Optional instance profile: if `InstanceRoleName` is set, an instance profile is created for that role and attached to the instance
- ✅ Optional user data: `UserDataBase64` is passed to the instance as-is
- ✅ Optional target group registration: if `TargetGroupArn` is set, the instance is registered with that target group on `TargetPort`, and deregistered when the stack is deleted

### EC2 Instance Parameters

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `ProjectName` | String | — | Project name used in the `Name` tag (required) |
| `InstanceBaseName` | String | `ec2` | Base name used in the `Name` tag |
| `environment` | String | `devl` | Deployment environment |
| `CiSuffix` | String | `""` | Optional CI suffix appended to the `Name` tag |
| `ImageId` | AWS::EC2::Image::Id (SSM) | Latest Amazon Linux 2023 | AMI ID, or SSM parameter holding one |
| `InstanceType` | String | `t3.micro` | EC2 instance type |
| `SubnetId` | AWS::EC2::Subnet::Id | — | Subnet to launch the instance in (required) |
| `SecurityGroupIds` | List<AWS::EC2::SecurityGroup::Id> | — | Security groups to attach (required) |
| `InstanceRoleName` | String | `""` | Name of an existing IAM role (not an ARN). Creates and attaches an instance profile when set |
| `UserDataBase64` | String | `""` | Base64-encoded user data (max 4096 characters, the CloudFormation parameter limit) |
| `TargetGroupArn` | String | `""` | ARN of an existing target group. Registers the instance when set |
| `TargetPort` | Number | `80` | Port the target group forwards to on the instance. Used only when `TargetGroupArn` is set |

### EC2 Instance Outputs

- `InstanceId` — EC2 instance ID
- `InstancePrivateIp` — Private IP address of the instance
- `InstanceProfileArn` — Instance profile ARN (only when `InstanceRoleName` is set)
- `TargetGroupArn` — Target group ARN the instance is registered with (only when `TargetGroupArn` is set)

All outputs are exported as `<stack-name>-<OutputName>`.

### Deploy

```bash
# Encode a user data script (macOS: base64 -i, Linux: base64 -w0)
USER_DATA=$(base64 -i userdata.sh | tr -d '\n')

aws cloudformation create-stack \
  --stack-name cfn-ec2-instance-dev \
  --template-body file://cloudformation/template.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=environment,ParameterValue=devl \
    ParameterKey=SubnetId,ParameterValue=subnet-0123456789abcdef0 \
    ParameterKey=SecurityGroupIds,ParameterValue=sg-0123456789abcdef0 \
    ParameterKey=InstanceRoleName,ParameterValue=my-ec2-role \
    ParameterKey=UserDataBase64,ParameterValue="$USER_DATA" \
    ParameterKey=InstanceBaseName,ParameterValue=web \
    ParameterKey=CiSuffix,ParameterValue=build-42 \
    ParameterKey=TargetGroupArn,ParameterValue=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-tg/0123456789abcdef \
    ParameterKey=TargetPort,ParameterValue=8080
```

With the example above, the instance is named `myproject-web-devl-us-east-1-build-42`.

Omit `InstanceRoleName`, `UserDataBase64`, and `TargetGroupArn` to launch an instance without an instance profile, user data, or target group registration. `CAPABILITY_IAM` is needed when an instance profile is created or a target group is set, since the registration custom resource creates an IAM role.

---

This repository also contains nested CloudFormation templates for deploying S3 buckets with security best practices and optional policy enforcement.

## Overview

This is a **nested stack template** designed to be invoked from a parent/root CloudFormation stack. Templates are stored in this repository and should be uploaded to an S3 bucket for reference by parent stacks.

## Template Files

### CloudFormation Templates

- **`templates/s3-bucket.yaml`** — Nested template for S3 bucket creation (versioning, public access blocking)
- **`templates/s3-bucket-policy.yaml`** — Optional nested template for S3 bucket policy (encryption enforcement, secure transport)

### Parameter Files

- **`parameters/parameters.json`** — Parameter values for development environment

## Template Features

### S3 Bucket Template (s3-bucket.yaml)

- ✅ Versioning enabled by default
- ✅ Public Access Blocking
- ✅ Smart bucket naming (project prefix, account ID, environment, region)
- ✅ Optional CI suffix support

### S3 Bucket Policy Template (s3-bucket-policy.yaml)

- ✅ Encryption enforcement on uploads
- ✅ Secure transport enforcement (HTTPS only)
- ✅ Optional/conditional policy rules

## Parameters

### S3 Bucket Parameters

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `ProjectName` | String | — | Project name to use as bucket prefix (required) |
| `BucketBaseName` | String | `cfn-bucket` | Base name for S3 bucket |
| `environment` | String | `devl` | Deployment environment (devl, stag, prod) |
| `CiSuffix` | String | `""` | Optional CI suffix to append to bucket name |

### S3 Bucket Policy Parameters

**Generic (Standalone) Mode:**

| Parameter | Type | Description |
| ----------- | ------ | ------------- |
| `BucketName` | String | Direct bucket name (use for any bucket) |

**Integrated Mode (with bucket template):**

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `ProjectName` | String | — | Project name (must match bucket template) |
| `BucketBaseName` | String | `cfn-bucket` | Base name (must match bucket template) |
| `environment` | String | `devl` | Environment (must match bucket template) |
| `CiSuffix` | String | `""` | CI suffix (must match bucket template) |

**Usage:** If `BucketName` is provided (non-empty), it takes precedence. Otherwise, the bucket name is constructed from ProjectName/BucketBaseName/environment/CiSuffix.

## Outputs

### S3 Bucket Template Outputs

- `S3BucketName` — S3 bucket name
- `S3BucketArn` — S3 bucket ARN

### S3 Bucket Policy Template Outputs

- `BucketName` — Bucket name with policy applied
- `PolicyStatus` — Policy application status (Applied)

## Usage

### 1. Upload Templates to S3

```bash
aws s3 cp templates/s3-bucket.yaml s3://your-cfn-bucket/templates/s3-bucket.yaml
aws s3 cp templates/s3-bucket-policy.yaml s3://your-cfn-bucket/templates/s3-bucket-policy.yaml
```

### 2. Reference from Parent Stack

In your parent/root CloudFormation template:

```yaml
S3BucketNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/s3-bucket.yaml
    Parameters:
      ProjectName: !Ref ProjectName
      BucketBaseName: cfn-bucket
      environment: !Ref Environment
      CiSuffix: !Ref CiSuffix
    Tags:
      - Key: Environment
        Value: !Ref Environment

S3PolicyNestedStack:
  Type: AWS::CloudFormation::Stack
  DependsOn: S3BucketNestedStack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/s3-bucket-policy.yaml
    Parameters:
      BucketName: !GetAtt S3BucketNestedStack.Outputs.S3BucketName
      ProjectName: ""
      BucketBaseName: cfn-bucket
      environment: !Ref Environment
      CiSuffix: !Ref CiSuffix
    Tags:
      - Key: Environment
        Value: !Ref Environment

Outputs:
  BucketName:
    Value: !GetAtt S3BucketNestedStack.Outputs.S3BucketName
  BucketArn:
    Value: !GetAtt S3BucketNestedStack.Outputs.S3BucketArn
```

### 3. Deploy Using AWS CLI

#### Option A: Deploy Bucket Only

```bash
# Development (without CI prefix)
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-dev \
  --template-body file://templates/s3-bucket.yaml \
  --parameters file://parameters/dev.json

# Staging (without CI prefix)
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-stag \
  --template-body file://templates/s3-bucket.yaml \
  --parameters file://parameters/staging.json

# Production (without CI prefix)
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-prod \
  --template-body file://templates/s3-bucket.yaml \
  --parameters file://parameters/prod.json
```

#### Option B: Deploy Bucket + Policy (Recommended)

```bash
# Deploy bucket first
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-dev \
  --template-body file://templates/s3-bucket.yaml \
  --parameters file://parameters/dev.json

# Wait for bucket to be created
aws cloudformation wait stack-create-complete --stack-name cfn-s3-bucket-dev

# Get the bucket name from stack outputs
BUCKET_NAME=$(aws cloudformation describe-stacks \
  --stack-name cfn-s3-bucket-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`S3BucketName`].OutputValue' \
  --output text)

#### Using Integrated Mode (with bucket template parameters)

```bash
aws cloudformation create-stack \
  --stack-name cfn-s3-policy-dev \
  --template-body file://templates/s3-bucket-policy.yaml \
  --parameters file://parameters/policy-dev.json
```

#### Using Generic Mode (standalone with direct bucket name)

```bash
aws cloudformation create-stack \
  --stack-name cfn-s3-policy-dev \
  --template-body file://templates/s3-bucket-policy.yaml \
  --parameters \
    ParameterKey=BucketName,ParameterValue=my-existing-bucket \
    ParameterKey=ProjectName,ParameterValue=""
```

#### Option C: Deploy with CI Suffix

```bash
# Development with CI suffix (e.g., for GitLab CI)
aws cloudformation create-stack \
  --stack-name cfn-s3-bucket-dev-ci \
  --template-body file://templates/s3-bucket.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=myproject \
    ParameterKey=BucketBaseName,ParameterValue=cfn-bucket \
    ParameterKey=environment,ParameterValue=devl \
    ParameterKey=CiSuffix,ParameterValue=$CI_PIPELINE_ID
```

## Bucket Naming Convention

The templates generate bucket names using the following pattern:

**Without CI Suffix:**

```bash
{ProjectName}-{BucketBaseName}-{AccountId}-{Environment}-{Region}
```

Example: `myproject-cfn-bucket-123456789012-devl-us-east-1`

**With CI Suffix:**

```bash
{ProjectName}-{BucketBaseName}-{AccountId}-{Environment}-{Region}-{CiSuffix}
```

Example: `myproject-cfn-bucket-123456789012-devl-us-east-1-pipeline-12345`

## Best Practices Implemented

- ✅ Versioning enabled by default
- ✅ Public access blocked by default
- ✅ Smart bucket naming with project prefix, account ID, environment, and region
- ✅ Optional CI suffix support for unique deployments
- ✅ Optional policy enforcement (encryption and secure transport)
- ✅ Export values for cross-stack references

## License

MIT
