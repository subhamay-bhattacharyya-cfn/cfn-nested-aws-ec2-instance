# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** that provides a reusable nested stack template for creating a single EC2 instance. The template is designed to be referenced by parent CloudFormation stacks via `TemplateURL`, and it also runs as a standalone stack in CI.

**Key characteristics:**

- Single nested CloudFormation template: `cloudformation/template.yaml`
- Instance `Name` tag built from project, base name, environment, region, and optional CI suffix
- IMDSv2 required on the instance
- Optional instance profile for an existing IAM role
- Optional Base64 user data, passed through as-is
- Optional registration with an existing target group, via a custom resource
- CI deployments (`CiSuffix` set) create their own VPC, public subnet, security group, and key pair
- Automated semantic versioning and releases
- AWS OIDC authentication for CI/CD deployments

## Project Structure

```text
cloudformation/
├── template.yaml                 # Nested template: EC2 instance (+ CI network, target group registration)
├── parameters.json               # Parameter values used by CI
└── stack-config.json             # Stack name, template file, and parameter file used by CI

.github/workflows/
├── ci.yaml                       # Loads stack config and calls the reusable CI build workflow
├── release.yaml                  # Semantic release on push to main
├── claude.yaml                   # Claude Code workflow
├── claude-code-review.yaml       # Claude Code review workflow
├── create-branch.yaml            # Auto-create feature branches from issues
├── notify.yaml                   # Notifications
└── setup-environments.yaml       # GitHub environment setup

scripts/plugins/
├── release.config.js             # Semantic-release configuration
└── (other release plugins)       # Commit analysis, notes generation, publish, prepare

.claude/
└── .skills/                      # Repository skills (see SKILLS_INDEX.md)

.devcontainer/
└── devcontainer.json             # Dev container setup (Node.js 20)

.releaserc.json                   # Semantic-release plugin list (branch: main)
package.json                      # Dependencies: semantic-release, commitizen
CHANGELOG.md                      # Generated release history
README.md                         # Template documentation and usage examples
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Lint the template

```bash
cfn-lint cloudformation/template.yaml
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat` and `fix` trigger releases.

## Key Architecture Concepts

### Nested Stack Pattern

This repo provides a **nested stack template**, referenced from parent/root CloudFormation stacks via `TemplateURL`. The template is self-contained and exports outputs for cross-stack references.

- **Parent stack** calls: `AWS::CloudFormation::Stack` with `TemplateURL` pointing to S3
- **Nested template** outputs values via the `Outputs` section
- Parent retrieves outputs via `!GetAtt NestedStack.Outputs.OutputKey`

### Instance Name Tag

```bash
{ProjectName}-{InstanceBaseName}-{Environment}-{AWS::Region}[-{CiSuffix}]
```

Example: `myproject-web-devl-us-east-1` (or `myproject-web-devl-us-east-1-build-42` with a CI suffix)

### Conditional Behavior

Conditions in `cloudformation/template.yaml`:

- `HasCiSuffix`: `CiSuffix` is set. Creates the network (VPC, internet gateway, public subnet, route table, security group), a key pair, and suffixes resource `Name` tags. Ignores `SubnetId`, `SecurityGroupIds`, and `KeyName`
- `HasInstanceRole`: `InstanceRoleName` is set. Creates an instance profile for the role
- `HasUserData`: `UserDataBase64` is set. Passed to the instance as user data
- `HasKeyName`: `KeyName` is set and `CiSuffix` is empty. Attaches an existing key pair
- `HasTargetGroup`: `TargetGroupArn` is set. Creates the target registration resources

A rule (`RequireExistingNetwork`) requires `SubnetId` when `CiSuffix` is empty.

### Target Group Registration

CloudFormation has no native resource for adding a target to an existing target group, so `template.yaml` uses a custom resource:

- `TargetRegistrationRole`: IAM role with `elasticloadbalancing:RegisterTargets` and `DeregisterTargets` on the target group, plus basic Lambda logging
- `TargetRegistrationFunction`: inline Python 3.12 Lambda that registers the instance on create, moves the registration on update when the target, port, or group changes, and deregisters on delete
- `TargetRegistration`: `AWS::CloudFormation::CustomResource` that passes `TargetGroupArn`, the instance ID, and `TargetPort`

Do not add an `AWS::ElasticLoadBalancingV2::TargetGroupAttachment` resource. It does not exist in CloudFormation.

### Parameter Conventions

- `Environment` is an `AllowedValues` list: `devl`, `test`, `prod`
- `UserDataBase64` must already be Base64 encoded and is limited to 4096 characters (the CloudFormation parameter limit)
- `CiSuffix` is limited to 30 characters, letters, numbers, and hyphens
- `SecurityGroupIds` is a `CommaDelimitedList`
- `cfn-lint` rule W1030 is ignored for the whole template because the empty defaults for `SubnetId` and `SecurityGroupIds` fail the ID patterns on the non-CI branch. Keep this in mind if you change those defaults

## Key Files to Understand

### `cloudformation/template.yaml`

**Purpose:** Creates one EC2 instance, with optional instance profile, user data, CI network, and target group registration

**Key inputs:**

- `ProjectName` (required): Project prefix used in the `Name` tag
- `InstanceBaseName` (default: `ec2`): Base name in the `Name` tag
- `Environment`: `devl`, `test`, or `prod`
- `CiSuffix`: Optional suffix. Switches the template into CI mode
- `SubnetId` and `SecurityGroupIds`: Required when `CiSuffix` is empty
- `InstanceRoleName`: Name (not ARN) of an existing IAM role
- `UserDataBase64`: Optional Base64 user data
- `TargetGroupArn` and `TargetPort`: Optional target group registration

**Key outputs:**

- `InstanceId`, `InstancePrivateIp`: Always present
- `InstanceProfileArn`: When `InstanceRoleName` is set
- `TargetGroupArn`: When `TargetGroupArn` is set
- `VpcId`, `PublicSubnetId`, `SecurityGroupId`, `KeyPairName`: When `CiSuffix` is set

The outputs do not declare `Export`, so parent stacks read them with `!GetAtt NestedStack.Outputs.<OutputName>`. Add an `Export` only if a cross-stack import is needed.

### `.github/workflows/ci.yaml`

**Triggered on:**

- Manual `workflow_dispatch`
- Pushes to `feature/**` and `bug/**` branches
- Pull requests to `main`

**Path filter:** Only runs if changes to `cloudformation/**` or `.github/workflows/ci.yaml`

**Phases:**

1. **Load config:** Reads `cloudformation/stack-config.json` for the stack name, template file, and parameter file
2. **CI build:** Calls the reusable workflow `subhamay-bhattacharyya-gha/cfn-ci-build-reusable-wf` with the `ci` environment. Validation, deployment, and cleanup are handled there
3. **Changelog and release:** On `main` only, generates `CHANGELOG.md` with git-cliff and creates the release

### `cloudformation/stack-config.json`

Single source of truth for CI: `stackName`, `templateFile`, and `parameterFile`. Update it if the template or parameter file moves.

### `.github/workflows/release.yaml`

**Triggered:** On push to main

**Release rules:**

- `feat:` → MINOR bump (0.1.0 → 0.2.0)
- `fix:` → PATCH bump (0.1.0 → 0.1.1)
- `BREAKING CHANGE:` → MAJOR bump (0.1.0 → 1.0.0)
- Other commits → no release

## Testing & Validation

**Lint and validate locally:**

```bash
cfn-lint cloudformation/template.yaml
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

`cfn-lint` may report informational notices (metadata context, cfn-guard checks such as inline policies and Lambda VPC placement). These are advisory. Errors must be fixed.

**Manual stack deployment (non-CI, existing network):**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nstd-ec2-devl \
  --parameter-overrides file://cloudformation/parameters.json \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

`CAPABILITY_IAM` is needed when an instance profile is created or a target group is set.

The CI workflow (`ci.yaml`) runs validation and deployment automatically, then cleans up.

## AWS Credentials & Environment Variables

The `ci` GitHub environment is passed to the reusable CI workflow. AWS access uses OIDC for keyless authentication. The GitHub OIDC provider must trust the role that the workflow assumes. The reusable workflow owns the exact variable names it reads, so check it before changing the environment configuration.

## Conventional Commits & Release Flow

This repo enforces conventional commits to drive semantic versioning:

```bash
npx cz commit
```

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying the Template

1. **Edit `cloudformation/template.yaml`**
2. **Update `cloudformation/parameters.json`** if parameters are added or renamed, and keep the README parameter table in sync
3. **Lint and validate** with `cfn-lint` and `aws cloudformation validate-template`
4. **Check the Name tag** still matches `{ProjectName}-{InstanceBaseName}-{Environment}-{AWS::Region}[-{CiSuffix}]` in both branches of the condition
5. **Create a PR** with a conventional commit message (e.g., `feat: add target group registration`)
6. **CI validates and deploys** automatically
7. **Merge to main** → release workflow creates the version tag and GitHub release

## Dev Container

Pre-configured with:

- Node.js 20
- GitHub Copilot extension

Use via VS Code: `code --remote-container-url <repo-url>`

## Current Branch

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
