# ecs-lab-infra

CloudFormation infrastructure for the ECS CI/CD lab.
All stacks are deployed automatically via CloudFormation GitSync.

## Repository structure

```
ecs-lab-infra/
├── templates/
│   ├── vpc.yaml          # Stack 1 — VPC, subnets, NAT GWs, VPC endpoints, security groups
│   ├── ecs.yaml          # Stack 2 — ECR, ALB, ECS cluster, task def, service, auto scaling, CodeDeploy
│   └── pipeline.yaml     # Stack 3 — OIDC role, S3 artifact bucket, EventBridge rule, CodePipeline
└── sync-configs/
    ├── vpc-sync.yaml      # GitSync config for vpc stack
    ├── ecs-sync.yaml      # GitSync config for ecs stack
    └── pipeline-sync.yaml # GitSync config for pipeline stack
```

## Stack deployment order

Stacks must be deployed in this order because each imports outputs from the previous:

```
vpc  →  ecs  →  pipeline
```

## Setup steps

### 1. Update placeholders in sync-configs/
Replace the following in all three sync-config files:
- `YOUR_ACCOUNT_ID` → your 12-digit AWS account ID
- `YOUR_GITHUB_USERNAME` → your GitHub username or org
- `YOUR_APP_REPO_NAME` → name of your app repo (e.g. ecs-lab-app)
- `YOUR_INFRA_REPO_NAME` → name of this repo (e.g. ecs-lab-infra)

### 2. Enable GitSync in the AWS Console (one-time)
1. Go to CloudFormation → GitSync
2. Connect your GitHub account (creates a CodeStar Connection)
3. For each stack, create a sync configuration pointing to the sync-config file

### 3. After stacks deploy — grab secrets for the app repo
From the pipeline stack Outputs in the AWS Console:
- `GitHubActionsRoleArn` → set as `AWS_ROLE_ARN` secret in the app repo
- `ArtifactBucketName`  → set as `PIPELINE_ARTIFACT_BUCKET` secret in the app repo

### 4. Push your app repo
The GitHub Actions workflow will build the image, push to ECR,
and the pipeline will automatically deploy it to ECS.
