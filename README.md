# S3-lab-infra

CloudFormation infrastructure for the Photo Gallery ECS lab.
All resources are provisioned in a single stack (`s3-lab-main`) deployed automatically via CloudFormation GitSync.

## Repository structure

```
S3-lab-infra/
├── main.yaml         # Single CloudFormation stack — all resources
└── main-sync.yaml    # GitSync configuration — links this repo to the CloudFormation stack
```

## What the stack provisions

| Resource | Details |
|----------|---------|
| VPC | Multi-AZ — 2 public subnets (ALB), 2 private subnets (ECS, RDS) |
| Security groups | ALB, ECS, RDS — least-privilege rules |
| Internet Gateway | Public subnet internet access |
| VPC Endpoints | ECR API, ECR DKR, S3 (Gateway), CloudWatch Logs, Secrets Manager |
| ECR Repository | Stores Docker images built by GitHub Actions |
| ECS Cluster + Service | Fargate, private subnets, auto scaling (1–4 tasks, CPU-based) |
| ECS Task Definition | Bootstrapped at `DesiredCount: 0` — CodeDeploy handles the first deployment |
| ALB | Public, HTTP port 80, routes to ECS tasks |
| RDS PostgreSQL | `db.t3.micro`, private subnet, password from Secrets Manager |
| Secrets Manager | Auto-generated DB password stored as JSON |
| S3 Image Bucket | Private — accessible only via CloudFront OAC |
| CloudFront Distribution | Price Class 200, OAC-restricted access to S3 |
| GitHub Actions OIDC Role | Allows `alpShema/S3-lab-app` on `main` to assume role — no long-lived credentials |
| CodeDeploy | Blue/green deployment, `TerminationWaitTimeInMinutes: 0` for fast deployments |
| CodePipeline | 2-stage: S3 source → CodeDeploy to ECS |
| EventBridge Rule | Triggers pipeline on ECR image push |
| IAM Roles | Task role, execution role, CodeDeploy role, pipeline role, GitHub Actions role |

## Setup steps

### 1. Connect GitHub to CloudFormation (one-time)
1. Go to **CloudFormation → Git sync** in the AWS Console
2. Click **Create stack with Git sync**
3. Connect your GitHub account (creates a CodeStar Connection)
4. Point the sync to `main-sync.yaml` in this repo

### 2. Stack deploys automatically
GitSync reads `main-sync.yaml` and deploys the `s3-lab-main` stack from `main.yaml`.
Any push to this repo triggers an automatic stack update.

### 3. Set GitHub variables in the app repo
Once the stack reaches `CREATE_COMPLETE`, go to CloudFormation → `s3-lab-main` → Outputs and set these in the **S3-lab-app** GitHub repo under **Settings → Secrets and variables → Actions → Variables**:

| Variable | CloudFormation Output |
|----------|-----------------------|
| `AWS_ROLE_ARN` | `GitHubActionsRoleArn` |
| `PIPELINE_ARTIFACT_BUCKET` | `ArtifactBucketName` |
| `AWS_REGION` | `eu-central-1` |
| `ECR_REPOSITORY` | `s3-lab-app` |

### 4. Push the app repo
The GitHub Actions workflow will build the image, push to ECR, resolve live stack outputs, and trigger the pipeline. CodeDeploy will perform the first blue/green deployment.

## Stack design decisions

**Single stack** — all resources in `main.yaml` instead of the original 3-stack design. Simpler to manage for a lab, avoids cross-stack export dependencies.

**`DesiredCount: 0`** — the ECS service is created with zero tasks. This prevents CloudFormation from waiting indefinitely for healthy tasks when ECR is empty on first deploy. CodeDeploy starts the first task when the pipeline runs.

**`TerminationWaitTimeInMinutes: 0`** — CodeDeploy terminates old (blue) tasks immediately after traffic shifts to green. Reduces deployment time from ~7 minutes to ~2 minutes.

**No NAT Gateway** — ECS tasks use VPC Endpoints for all AWS service calls (ECR, S3, CloudWatch, Secrets Manager). This avoids ~$32/month NAT Gateway cost. CloudFront image serving goes through the OAC — no NAT needed.

## Stack outputs used by the app

| Output Key | Used for |
|------------|---------|
| `GitHubActionsRoleArn` | GitHub Actions OIDC role assumption |
| `ArtifactBucketName` | GitHub Actions uploads `artifacts.zip` here |
| `TaskExecutionRoleArn` | Injected into `taskdef.json` by the workflow |
| `TaskRoleArn` | Injected into `taskdef.json` by the workflow |
| `ImageBucketName` | Injected into `taskdef.json` as `S3_BUCKET` |
| `CloudFrontUrl` | Injected into `taskdef.json` as `CLOUDFRONT_URL` |
| `CloudFrontDistributionId` | Injected into `taskdef.json` |
| `RdsEndpoint` | Injected into `taskdef.json` as `DB_HOST` |
| `DBSecretArn` | Injected into `taskdef.json` secrets `valueFrom` |
| `AlbEndpoint` | URL to access the running application |
