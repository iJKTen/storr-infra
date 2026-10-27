# Storr Infrastructure

CloudFormation templates for all AWS infrastructure powering Storr  - a Shopify store builder platform that enables merchants to create AI-generated storefronts with live preview environments.

## Table of Contents

- [Architecture Overview](#architecture-overview)
  - [SSM Parameter Contract](#ssm-parameter-contract)
- [Stack-by-Stack Guide](#stack-by-stack-guide)
  - [00-role.yaml  - Git Sync IAM Role](#00-roleyaml----git-sync-iam-role-manually-deployed)
  - [05-foundation.yaml  - ECR, CodeArtifact, KMS, S3](#05-foundationyaml----ecr-codeartifact-kms-s3)
  - [10-network.yaml  - VPC, Subnets, Security Groups](#10-networkyaml----vpc-subnets-security-groups)
  - [20-data.yaml  - DynamoDB, Cognito](#20-datayaml----dynamodb-cognito)
  - [30-storage.yaml  - EFS, S3 Product Images](#30-storageyaml----efs-s3-product-images)
  - [35-config.yaml  - Secrets Manager and SSM Config](#35-configyaml----secrets-manager-and-ssm-config)
  - [40-services.yaml  - ECS Fargate, ALB, DNS](#40-servicesyaml----ecs-fargate-alb-dns)
  - [45-api.yaml  - Lambda, API Gateway, Custom Domain](#45-apiyaml----lambda-api-gateway-custom-domain)
  - [50-frontend.yaml  - S3, CloudFront, DNS](#50-frontendyaml----s3-cloudfront-dns)
  - [55-cicd.yaml  - CodePipeline, CodeBuild](#55-cicdyaml----codepipeline-codebuild)
  - [60-observability.yaml  - SNS, CloudWatch Alarms, Dashboard](#60-observabilityyaml----sns-cloudwatch-alarms-dashboard)
- [How to Deploy](#how-to-deploy)
  - [Prerequisites](#prerequisites)
  - [Steps (per stack)](#steps-per-stack)
  - [Deployment Order](#deployment-order)
  - [Seed ECR Images](#seed-ecr-images)
  - [Secret Setup for 35-config](#secret-setup-for-35-config)
- [Deploying Application Code](#deploying-application-code)
  - [Phase 1: Automatic](#phase-1-automatic-runs-in-parallel-when-55-cicd-is-created)
  - [Phase 2: Verification](#phase-2-verification)
- [SSM Parameter Quick Reference](#ssm-parameter-quick-reference)
- [End-to-End Data Flow](#end-to-end-data-flow)
- [Teardown](#teardown)

---

## Architecture Overview

Storr uses a **multi-repo split**:
- **storr-api**  - Go Lambda API for store management, OAuth, and runtime orchestration
- **storr-app**  - React frontend dashboard for managing stores
- **storr-runtime**  - Docker runtime environment for live studio previews
- **storr-pipeline**  - Docker pipeline for theme builds and deployments
- **storr-agent**  - AI agent for code generation and theme customization
- **storr-prompt**  - Prompt library for AI interactions
- **storr-figma-parser**  - Figma design file parser for theme generation
- **eve**  - Shopify theme framework (shared foundation)
- **storr-infra** (this repo)  - raw CloudFormation YAML templates for all AWS resources

This repository consolidates infrastructure that was previously split across 6 separate repos (storr-cert-infra, storr-studio-infra, storr-api-infra, storr-app-infra, storr-runtime-infra, storr-figma-parser-infra).

> **Certificate exception:** the `storr.build` ACM certificate is no longer owned here. It is a shared resource consumed by both this app and the separate `storr-saas-infra` marketing site, so it lives in the standalone `storr-cert-infra` stack and is read via the SSM parameter `/storr/acm/cert-arn`. Placing it in this app stack would couple the SaaS site to the app. See `docs/adr/0005-consume-shared-acm-certificate.md`.

Templates are deployed via **CloudFormation Git Sync** from the AWS Console. You connect a stack to this repo, point it at a deployment file, and AWS automatically deploys changes when you push to the branch.

### SSM Parameter Contract

Stacks communicate through SSM parameters instead of CloudFormation `Export`/`!ImportValue`. This avoids cross-stack coupling  - you can update or replace a stack without breaking dependent stacks.

**Producer** (`05-foundation.yaml`):
```yaml
KmsKeyArnParam:
  Type: AWS::SSM::Parameter
  Properties:
    Name: /storr/kms/key-arn
    Type: String
    Value: !GetAtt KmsKey.Arn
```

**Consumer** (`40-services.yaml`):
```yaml
StudioExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    Policies:
      - PolicyName: SsmSecretAccess
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action: ssm:GetParameters
              Resource: !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter/storr/*'
```

The `{{resolve:ssm:...}}` syntax resolves at deploy time, pulling the current value from SSM.

---

## Stack-by-Stack Guide

Deploy stacks in numeric order. Each stack must reach `CREATE_COMPLETE` before deploying the next.

### 00-role.yaml  - Git Sync IAM Role (Manually Deployed)

**What it creates:**
- `StorrGitSyncRole` IAM role with trust policy for `cloudformation.sync.codeconnections.amazonaws.com` and `cloudformation.amazonaws.com`
- 4 customer-managed IAM policies: `StorrNetworkAndComputePolicy`, `StorrDataAndAppPolicy`, `StorrDeliveryAndObservabilityPolicy`, `StorrCoreServicesPolicy`
- 1 inline policy `StorrScopedResources` for SSM parameters (scoped to `/storr/*`) and CloudWatch Logs

**How the app uses it:**
- All Storr CloudFormation stacks use this role for Git Sync deployments
- Customer-managed policies allow up to 6,144 characters each, avoiding the 10,240-character aggregate inline policy limit
- `StorrCoreServicesPolicy` enumerates specific CloudFormation, S3, IAM, and CodePipeline actions (no AWS-managed full-access policies)

**Deployment:** This stack is deployed outside Git Sync, because the role is a prerequisite for Git Sync itself. Manage its entire lifecycle (create, update, delete) with the shared external `CloudFormationGitSyncRole` from `zstack-global-infra`, never with `StorrGitSyncRole` as the stack's own service role. That self-reference makes deletion fail: CloudFormation deletes the role mid-operation and cannot finish, and the console reuses the stored (now-deleted) role on every retry. The external role is not one of this stack's resources, so it survives every operation.

> **The create-time role choice is what prevents the teardown failure, and no change to `00-role.yaml` can control it.** The service role is set by the caller and stored on the stack; the console Delete / Retry delete buttons reuse that stored role. Create the stack with the external role so deletion is safe even from the console button:
> - **CLI:** `aws cloudformation deploy --stack-name <role stack> --template-file 00-role.yaml --capabilities CAPABILITY_NAMED_IAM --role-arn arn:aws:iam::<account>:role/CloudFormationGitSyncRole`
> - **Console:** Create stack > Upload a template file, then on the **Permissions** step select `CloudFormationGitSyncRole` (not `StorrGitSyncRole`).
>
> If the stack was instead created with `StorrGitSyncRole` as its service role, you must delete it with the `--role-arn` override described in [Teardown](#teardown); the console Delete button will fail.

See ADR-0007. The deployment file `00-deployment-role.yaml` exists for consistency but is used only for manual console deploys, not Git Sync.

**SSM parameters written:** none

---

### 05-foundation.yaml  - ECR, CodeArtifact, KMS, S3

**What it creates:**
- 2 ECR repositories (storr-studio, storr-pipeline) with KMS encryption, scan-on-push, and lifecycle policies (expire untagged after 1 day, keep the last 5 images tagged `latest`)
- CodeArtifact domain `jaik` with two repositories: `npmjs-store` (upstream to public npm) and `storr` (for private packages)
- 1 KMS key with alias `storr` for encrypting ECR images (the eve-source S3 bucket does not use this key, see below)
- S3 bucket for the Eve Shopify theme framework source code (logical ID `EveSourceBucket`, physical name auto-generated; versioned, SSE-S3/AES256 encrypted, auto-empty on delete)

**How the app uses it:**
- The shared `storr.build` certificate (owned by `storr-cert-infra`, read from `/storr/acm/cert-arn`) secures all Storr app domains (app.storr.build, api.storr.build, *.preview.storr.build); this stack does not create it
- Studio and pipeline Docker images are pushed to ECR repos and pulled by ECS tasks at startup
- CodeArtifact hosts private npm packages shared across Storr repos
- The eve-source bucket stores a zipped copy of the Eve theme framework, downloaded by pipeline containers during theme builds
- The KMS key encrypts container images at rest (the eve-source S3 bucket is encrypted with SSE-S3/AES256, not the KMS key)

**SSM parameters written:**

| Parameter | Description |
|-----------|-------------|
| `/storr/acm/hosted-zone-id` | Route 53 hosted zone ID |
| `/storr/kms/key-arn` | KMS key ARN |
| `/storr/ecr/studio-repo-uri` | Studio ECR repo URI |

> `/storr/acm/cert-arn` is **read** by this stack's consumers but **written** by `storr-cert-infra`, not here.
| `/storr/ecr/studio-repo-arn` | Studio ECR repo ARN |
| `/storr/ecr/pipeline-repo-uri` | Pipeline ECR repo URI |
| `/storr/ecr/pipeline-repo-arn` | Pipeline ECR repo ARN |
| `/storr/codeartifact/domain-name` | CodeArtifact domain name |
| `/storr/codeartifact/domain-arn` | CodeArtifact domain ARN |
| `/storr/codeartifact/storr-repo-arn` | Storr CodeArtifact repository ARN |
| `/storr/codeartifact/storr-repo-endpoint` | Storr CodeArtifact npm endpoint |
| `/storr/s3/eve-source-bucket-name` | Eve source S3 bucket name |
| `/storr/s3/eve-source-bucket-arn` | Eve source S3 bucket ARN |

---

### 10-network.yaml  - VPC, Subnets, Security Groups

**What it creates:**
- VPC with `10.0.0.0/16` CIDR, DNS hostnames enabled
- 2 public subnets in AZ-a and AZ-b for ALB and Fargate tasks
- Internet Gateway and public route table
- 3 security groups: ALB (443, 80 inbound), Fargate (9292, 8080 from ALB), EFS (2049 from Fargate)

**How the app uses it:**
- The ALB sits in public subnets to receive internet traffic for preview environments
- ECS Fargate tasks (studio and pipeline containers) run in public subnets with outbound internet access
- The EFS mount target lives in AZ-a only (single-AZ to reduce cost)
- Security groups enforce that only the ALB can reach Fargate tasks on ports 8080 (studio) and 9292 (preview runtime), and only Fargate tasks can access EFS

**Reads from:** nothing (first networking stack)

**SSM parameters written:**

| Parameter | Description |
|-----------|-------------|
| `/storr/vpc/id` | VPC ID |
| `/storr/subnet/public-a` | Public subnet A |
| `/storr/subnet/public-b` | Public subnet B |
| `/storr/sg/alb` | ALB security group |
| `/storr/sg/fargate` | Fargate security group |
| `/storr/sg/efs` | EFS security group |

---

### 20-data.yaml  - DynamoDB, Cognito

**What it creates:**
- Cognito UserPool with email-based authentication, auto-verified email, and password policy (8+ chars, uppercase, lowercase, numbers)
- Cognito UserPoolClient with admin password auth and refresh token flows
- 4 DynamoDB tables (all PAY_PER_REQUEST with SSE enabled, DeletionPolicy: Retain):
  - **stores**: userId (HASH), storeId (RANGE)  - stores store metadata and configuration
  - **oauth-states**: state (HASH) with TTL  - temporary OAuth state tokens for Figma and Shopify flows
  - **products**: storeId (HASH), productId (RANGE) with GSI `shopifyId-index` (storeId HASH, shopifyId RANGE)  - product catalog synced from Shopify
  - **webhook-events**: eventId (HASH) with TTL  - Shopify webhook event log

**How the app uses it:**
- Cognito authenticates users and issues JWT tokens for the API
- The stores table holds store configuration, theme settings, and deployment status
- The oauth-states table stores temporary state tokens during OAuth flows with Figma and Shopify (TTL auto-expires after 15 minutes)
- The products table caches Shopify product data to avoid repeated API calls
- The webhook-events table logs Shopify webhook deliveries for debugging and replay

**Reads from:** nothing

**SSM parameters written:**

| Parameter | Description |
|-----------|-------------|
| `/storr/cognito/user-pool-id` | Cognito UserPool ID |
| `/storr/cognito/user-pool-arn` | Cognito UserPool ARN |
| `/storr/cognito/client-id` | Cognito UserPoolClient ID |
| `/storr/dynamodb/stores-table` | Stores table name |
| `/storr/dynamodb/stores-table-arn` | Stores table ARN |
| `/storr/dynamodb/oauth-states-table` | OAuth states table name |
| `/storr/dynamodb/oauth-states-table-arn` | OAuth states table ARN |
| `/storr/dynamodb/products-table` | Products table name |
| `/storr/dynamodb/products-table-arn` | Products table ARN |
| `/storr/dynamodb/webhook-events-table` | Webhook events table name |
| `/storr/dynamodb/webhook-events-table-arn` | Webhook events table ARN |

---

### 30-storage.yaml  - EFS, S3 Product Images

**What it creates:**
- EFS filesystem (encrypted, elastic throughput, lifecycle policy to IA after 30 days, DeletionPolicy: Retain)
- EFS mount target in public subnet A (single-AZ to reduce cost)
- S3 bucket for product images with CORS configured for uploads from `https://app.storr.build`, public access blocked, AES256 encryption, versioning enabled
- Lifecycle rules: abort incomplete multipart uploads after 1 day, expire orphaned images after 30 days (scoped to the `stores/` prefix only)
- Lambda-backed custom resource to empty the product-images bucket on stack deletion

**How the app uses it:**
- EFS stores theme source code, build artifacts, and runtime files for each store's studio environment
- Each studio environment gets an isolated EFS access point created by the API Lambda at runtime
- The product-images bucket stores user-uploaded product images for Shopify stores
- CORS allows the frontend to generate presigned upload URLs and upload directly to S3

**Reads from:** `10-network.yaml` (subnets, EFS security group)

**SSM parameters written:**

| Parameter | Description |
|-----------|-------------|
| `/storr/efs/filesystem-id` | EFS filesystem ID |
| `/storr/efs/filesystem-arn` | EFS filesystem ARN |
| `/storr/s3/product-images-bucket-name` | Product images S3 bucket name |
| `/storr/s3/product-images-bucket-arn` | Product images S3 bucket ARN |

---

### 35-config.yaml  - Secrets Manager and SSM Config

**What it creates:**
- 4 Secrets Manager secrets (KMS-encrypted via the Storr KMS key):
  - `storr/figma`: Figma OAuth credentials (client_id, client_secret)
  - `storr/shopify`: Shopify OAuth credentials (client_id, client_secret)
  - `storr/anthropic`: Anthropic API key for AI code generation
  - `storr/runtime-auth`: Shared auth token between API and runtime containers
- 4 SSM parameters for non-sensitive config (redirect URIs, LLM model, LLM provider)
- 4 SSM parameters storing secret ARNs for downstream consumption

**How the app uses it:**
- The API Lambda reads secret ARNs from SSM, then fetches actual credentials from Secrets Manager at runtime
- ECS task definitions reference the runtime-auth secret ARN for container-level secret injection
- The Anthropic secret is used by the agent service to call Claude for theme code generation
- Non-sensitive config (redirect URIs, LLM model/provider) is read directly from SSM

**Important:** Secrets are created with placeholder values via `GenerateSecretString`. After deploying this stack, update each secret in the Secrets Manager console with real credentials. This approach avoids storing secrets in deployment files or CloudFormation parameters.

**Reads from:** `05-foundation.yaml` (KMS key ARN via SSM)

**SSM parameters written:**

| Parameter | Description |
|-----------|-------------|
| `/storr/figma-redirect-uri` | Figma OAuth redirect URI |
| `/storr/shopify-redirect-uri` | Shopify OAuth redirect URI |
| `/storr/llm-model` | LLM model name |
| `/storr/llm-provider` | LLM provider |
| `/storr/secrets/figma-arn` | Figma Secrets Manager secret ARN |
| `/storr/secrets/shopify-arn` | Shopify Secrets Manager secret ARN |
| `/storr/secrets/anthropic-arn` | Anthropic Secrets Manager secret ARN |
| `/storr/secrets/runtime-auth-arn` | Runtime auth Secrets Manager secret ARN |

---

### 40-services.yaml  - ECS Fargate, ALB, DNS

**What it creates:**
- ECS Fargate cluster (Container Insights disabled, see note below)
- Internet-facing ALB with HTTPS (port 443) and HTTP to HTTPS redirect
- WAFv2 WebACL associated with the ALB (AWS Managed Rules: CommonRuleSet, KnownBadInputsRuleSet, plus IP-based rate limiting at 2000 requests per 5 minutes)
- HTTPS listener with default 503 response (preview environments register dynamically)
- Route 53 alias A and AAAA records pointing `*.preview.storr.build` to the ALB
- 3 CloudWatch log groups: /ecs/storr-studio, /ecs/storr-pipeline, /ecs/storr-preview-runtime (7-day retention)
- IAM execution role and task role for studio and pipeline containers (scoped to EFS, S3, SSM, Secrets Manager, KMS, logs)
- Lambda-backed custom resource to clean up runtime-created ECS tasks, ALB listener rules, and target groups on stack deletion
- 2 task definitions (studio: 512/1024, pipeline: 1024/4096)

**Prerequisite:** Docker images must exist in ECR before deploying this stack, because the task definitions reference `storr-studio:latest` and `storr-pipeline:latest`. See the [Seed ECR Images](#seed-ecr-images) section below.

**How the app uses it:**
- The ALB receives traffic for preview environments at `<store-id>.preview.storr.build`
- The API Lambda dynamically creates target groups, listener rules, and ECS tasks for each store
- Studio containers run on-demand when a user opens a studio environment (auto-shutdown after 30 minutes of inactivity)
- Pipeline containers run one-off tasks to build and deploy Shopify themes
- EFS is mounted at `/home/node/app` in studio containers to persist theme files across container restarts

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `IdleTimeoutMinutes` | `30` | Minutes of inactivity before auto-shutdown of studio environments (5-120) |

**Container Insights:** Disabled to save ~$11/month in CloudWatch custom metric costs. Container Insights generates per-task CPU, memory, and network metrics in the `ECS/ContainerInsights` namespace, which are billed at $0.30/metric/month. Lambda and ALB metrics from the `60-observability` dashboard are unaffected. To re-enable, change `containerInsights` from `disabled` to `enabled` on the `EcsCluster` resource in `40-services.yaml`.

**Reads from:** `05-foundation.yaml` (cert ARN, ECR repos, hosted zone ID), `10-network.yaml` (subnets, security groups), `30-storage.yaml` (EFS filesystem ID), `35-config.yaml` (runtime-auth secret ARN)

**SSM parameters written:**

| Parameter | Description |
|-----------|-------------|
| `/storr/ecs/cluster-name` | ECS cluster name |
| `/storr/ecs/cluster-arn` | ECS cluster ARN |
| `/storr/alb/arn` | ALB ARN |
| `/storr/alb/dns-name` | ALB DNS name |
| `/storr/alb/listener-arn` | HTTPS listener ARN |
| `/storr/alb/full-name` | ALB full name (for CloudWatch metrics) |
| `/storr/ecs/fargate-sg` | Fargate security group |
| `/storr/ecs/subnet-ids` | Comma-separated subnet IDs |
| `/storr/iam/studio-execution-role-arn` | Studio execution role ARN |
| `/storr/iam/studio-task-role-arn` | Studio task role ARN |
| `/storr/ecs/studio-task-def-arn` | Studio task definition ARN |
| `/storr/ecs/pipeline-task-def-arn` | Pipeline task definition ARN |

---

### 45-api.yaml  - Lambda, API Gateway, Custom Domain

**What it creates:**
- IAM role for the API Lambda with least-privilege permissions (DynamoDB, S3, Cognito, ECS, EFS, ALB, EC2 network interfaces, SSM, Secrets Manager, KMS, IAM PassRole, CloudWatch Logs)
- CloudWatch log group `/aws/lambda/storr-api` (14-day retention)
- Lambda-backed custom resource to seed a placeholder zip in the eve-source S3 bucket on first deploy (auto-seeded, idempotent, skips upload if real code already exists, cleans up on stack deletion)
- Lambda function `storr-api` (Go, provided.al2023, arm64, 512 MB, 60s timeout)
- API Gateway HTTP API with CORS configured for `https://app.storr.build`
- HTTP API integration using PayloadFormatVersion 1.0 (AWS Lambda proxy v1 format)
- Custom domain `api.storr.build` with Route 53 alias A and AAAA records

**How the app uses it:**
- The frontend calls the API at `https://api.storr.build` for all store operations, OAuth flows, and studio lifecycle management
- The Lambda function handles authentication, orchestrates ECS tasks, manages ALB routing, and performs CRUD on DynamoDB
- The Lambda reads secret ARNs from SSM parameters, then fetches credentials from Secrets Manager at runtime
- The API creates EFS access points, target groups, and listener rules dynamically at runtime
- CORS is handled by API Gateway, not the Lambda function

**Important:** The initial deployment uses a placeholder Lambda zip (`storr-api-placeholder.zip`) auto-seeded by a custom resource into the eve-source bucket on first deploy. The real Lambda code is deployed via the CI/CD pipeline after the stack is created. The placeholder zip is automatically deleted when the stack is torn down.

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `CorsOrigin` | `https://app.storr.build` | Allowed CORS origin |
| `DomainName` | `api.storr.build` | Custom domain for the API |
| `BatchUpdateMaxProducts` | `25` | Maximum number of products allowed in a single batch update request |

**Reads from:** all upstream stacks (DynamoDB table names/ARNs, Cognito, ECS cluster, subnets, security groups, EFS, S3 buckets, secret ARNs from Secrets Manager via SSM)

**SSM parameters written:**

| Parameter | Description |
|-----------|-------------|
| `/storr/api/url` | API URL (https://api.storr.build) |
| `/storr/api/function-name` | Lambda function name |
| `/storr/api/function-arn` | Lambda function ARN |

---

### 50-frontend.yaml  - S3, CloudFront, DNS

**What it creates:**
- S3 bucket for the React SPA (versioned, encrypted, public access blocked, 30-day noncurrent version expiry)
- Lambda-backed custom resource to empty the website bucket on stack deletion
- CloudFront Origin Access Control (OAC) and bucket policy allowing `cloudfront.amazonaws.com` with source ARN condition, plus a `DenyInsecureTransport` statement that rejects non-HTTPS requests
- CloudFront distribution with HTTPS-only, HTTP/2+3, SPA routing (403/404 to /index.html), CachingOptimized policy
- ResponseHeadersPolicy with HSTS, nosniff, frame DENY, strict-origin-when-cross-origin referrer policy, XSS protection, and CSP
- CSP: `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://api.storr.build; frame-ancestors 'none'; base-uri 'self'; form-action 'self';`
- Route 53 alias A and AAAA records pointing `app.storr.build` to CloudFront

**How the app uses it:**
- The React frontend is served from CloudFront with edge caching
- All routes are handled client-side (React Router), CloudFront rewrites all non-asset paths to /index.html
- The CSP allows API calls to `https://api.storr.build` and inline styles for React (common in bundled SPAs)
- The bucket is never publicly accessible, CloudFront is the only reader

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `DomainName` | `app.storr.build` | Fully qualified domain name for the frontend |
| `ApiDomain` | `api.storr.build` | API domain used in CSP connect-src header |

**Reads from:** `storr-cert-infra` (cert ARN via `/storr/acm/cert-arn`), `05-foundation.yaml` (hosted zone ID)

**SSM parameters written:**

| Parameter | Description |
|-----------|-------------|
| `/storr/cloudfront/distribution-id` | CloudFront distribution ID |
| `/storr/cloudfront/domain-name` | CloudFront domain name |
| `/storr/s3/website-bucket-name` | Website S3 bucket name |

---

### 55-cicd.yaml  - CodePipeline, CodeBuild

**What it creates:**
- S3 artifact bucket for pipeline artifacts (SSE-S3, 7-day expiry, auto-empty on delete)
- 6 CodePipeline pipelines (GitHub via CodeStar Connection). Stage counts vary per pipeline:
  1. **Storr API** (Source, Build, Deploy): Go build (GOARCH=arm64) to bootstrap.zip, deploy via Lambda UpdateFunctionCode
  2. **Storr App** (Source, Build, Deploy): npm build (Vite), S3 sync with Cache-Control headers, CloudFront invalidation
  3. **Storr Runtime** (Source, Build): Docker build, push to ECR storr-studio:latest
  4. **Storr Pipeline-Image** (Source, Build): 4-source Docker build (storr-pipeline, storr-agent, storr-prompt, storr-figma-parser), push to ECR storr-pipeline:latest
  5. **Eve-Source** (Source, Deploy): upload Eve theme framework zip to S3 eve-source bucket via S3 deploy action
  6. **Figma-Parser** (Source, Build): npm build, publish to CodeArtifact
- IAM roles for each CodeBuild project (least privilege, scoped to specific S3 buckets, ECR repos, Lambda functions, CloudFront distributions)
- IAM roles for each CodePipeline (one per pipeline). All CodePipeline and CodeBuild service-role trust policies include an `aws:SourceAccount` condition to guard against confused-deputy attacks

**How the app uses it:**
- Pushing to `main` in any of the application source repos triggers the corresponding pipeline automatically (6 pipelines across 8 distinct source repos; the Pipeline-Image pipeline watches 4 of them)
- The API pipeline builds the Go binary for ARM64, zips it, and updates the Lambda function code
- The App pipeline builds the Vite bundle, sets Cache-Control headers (HTML: no-cache, assets: immutable), syncs to S3, and invalidates CloudFront
- The Runtime and Pipeline-Image pipelines build and push Docker images to ECR with `:latest` and commit SHA tags
- The Eve-Source pipeline zips the Eve theme framework and uploads it to S3 for pipeline containers to download during theme builds
- The Figma-Parser pipeline builds and publishes the npm package to CodeArtifact for use as a dependency in other Storr repos

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `GitHubOwner` | `iJKTen` | GitHub organization or user |
| `GitHubBranch` | `main` | Branch to trigger pipelines |

**Reads from:** all upstream stacks (ECR repos, S3 buckets, Lambda function names, CloudFront distribution ID)

**SSM parameters written:** none

---

### 60-observability.yaml  - SNS, CloudWatch Alarms, Dashboard

**What it creates:**
- SNS topic (KMS-encrypted) with email subscription for alarm notifications
- 2 CloudWatch alarms with SNS notification actions:
  - **LambdaErrorAlarm**: triggers when the API Lambda has any errors (Errors > 0 over 5 minutes)
  - **Alb5xxAlarm**: triggers when the ALB target 5xx count exceeds 5 over 5 minutes
- CloudWatch dashboard (name auto-generated by CloudFormation) with 3 widgets:
  - Lambda metrics: Invocations, Errors, Duration for storr-api
  - ALB metrics: RequestCount, HTTPCode_Target_5XX_Count
  - DynamoDB metrics: ConsumedReadCapacityUnits, ConsumedWriteCapacityUnits for stores and products tables

**How the app uses it:**
- Both alarms send notifications to the SNS topic when entering ALARM or returning to OK state
- The email subscriber receives alarm notifications at the address specified by the `AlertEmail` parameter
- The dashboard provides a single-pane view of API health, ALB traffic, and DynamoDB usage
- CloudWatch Logs from ECS tasks and Lambda are viewed separately (no log groups created here, those are in 40-services and 45-api)

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AlertEmail` | (required) | Email address for alarm notifications |

**Reads from:** `45-api.yaml` (Lambda function name), `40-services.yaml` (ALB full name), `20-data.yaml` (DynamoDB table names)

**SSM parameters written:** none

---

## How to Deploy

### Prerequisites
- AWS account with CloudFormation Git Sync enabled
- This repo connected as a Git source in AWS
- `StorrGitSyncRole` deployed from `00-role.yaml` (manually deployed, see below)
- Route 53 hosted zone for `storr.build` (zone ID required as a parameter in `05-foundation.yaml`)
- The `storr-cert-infra` stack deployed, which owns the shared `storr.build` ACM certificate and publishes `/storr/acm/cert-arn` (consumed by `40-services.yaml`, `45-api.yaml`, and `50-frontend.yaml`)
- The `zstack-global-infra` stack deployed, which exports the GitHub CodeStar Connection ARN as `GlobalResourcesStack-GitHubConnectionArn` (used by all 6 pipelines in `55-cicd.yaml` via `!ImportValue`)

### Steps (per stack)

1. Open **AWS CloudFormation** in the console
2. Click **Create stack** > **Sync from Git**
3. Select this repository and branch (`main`)
4. Set the **deployment file path** (e.g., `05-deployment-foundation.yaml`)
5. Name the stack (e.g., `storr-foundation`)
6. Review and create
7. **Wait for `CREATE_COMPLETE`** before deploying the next stack

### Deployment Order

```
storr-cert-infra (shared cert) → 00-role (manual) → 05-foundation → 10-network → 20-data → 30-storage → 35-config → [seed ECR images] → 40-services → 45-api → 50-frontend → 55-cicd → 60-observability
```

`storr-cert-infra` is a separate repo/stack; it must exist before `40-services`, `45-api`, and `50-frontend` resolve `/storr/acm/cert-arn`. Deploy it first.

### Seed ECR Images

The `40-services` stack creates ECS task definitions that reference `storr-studio:latest` and `storr-pipeline:latest` in ECR. These images must exist before deploying the stack. Push placeholder images once before the first deploy:

```bash
# Authenticate with ECR
AWS_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com

# Push a minimal placeholder to both repos
for repo in storr-studio storr-pipeline; do
  docker pull public.ecr.aws/docker/library/alpine:3.20
  docker tag public.ecr.aws/docker/library/alpine:3.20 $AWS_ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com/$repo:latest
  docker push $AWS_ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com/$repo:latest
done
```

After `55-cicd` is deployed, the `storr-runtime` and `storr-pipeline` CodePipelines auto-trigger and replace the placeholder images with real ones.

### Secret Setup for 35-config

After deploying `35-config`, update each secret in the **Secrets Manager console** with real credentials:
- `storr/figma`: Set `client_id` and `client_secret` (from Figma Developer settings)
- `storr/shopify`: Set `client_id` and `client_secret` (from Shopify Partner Dashboard)
- `storr/anthropic`: Set `api_key` (from Anthropic Console)
- `storr/runtime-auth`: The `token` value is auto-generated (64 chars) and can be used as-is

The deployment file (`35-deployment-config.yaml`) only contains non-sensitive config (redirect URIs, LLM model/provider). No secret values are stored in deployment files or CloudFormation parameters.

---

## Deploying Application Code

Once all 11 infrastructure stacks are deployed, the application code needs to be built and deployed. When `55-cicd` is created, all 6 CodePipelines auto-trigger by picking up the latest commit from `main` in each repo. The first-time deployment has a dependency order.

### Phase 1: Automatic (runs in parallel when 55-cicd is created)

All 6 pipelines trigger simultaneously:

| Pipeline | What it does |
|----------|-------------|
| **eve** | Zips the Eve theme framework source and uploads `eve-source.zip` to S3 |
| **storr-runtime** | Builds Docker image, pushes to ECR `storr-studio` |
| **storr-pipeline** | Builds Docker image (bundles storr-agent, storr-prompt, storr-figma-parser), pushes to ECR `storr-pipeline` |
| **storr-figma-parser** | Builds and publishes the npm package to CodeArtifact |
| **storr-api** | Builds Go binary (arm64), deploys to Lambda (replaces placeholder) |
| **storr-app** | Builds React SPA (Vite), deploys to S3, invalidates CloudFront |

### Phase 2: Verification

- Confirm `app.storr.build` loads the React frontend
- Confirm `api.storr.build` returns a response
- Test a studio launch to verify ECS tasks start correctly and the preview environment loads at `<store-id>.preview.storr.build`

---

## SSM Parameter Quick Reference

All 59 parameters written across all stacks (`/storr/acm/cert-arn` is written by the separate `storr-cert-infra` stack, not counted here):

### 05-foundation (12 params)
| Parameter | Source |
|-----------|--------|
| `/storr/acm/hosted-zone-id` | Route 53 hosted zone ID |
| `/storr/kms/key-arn` | KMS key ARN |
| `/storr/ecr/studio-repo-uri` | Studio ECR repo URI |
| `/storr/ecr/studio-repo-arn` | Studio ECR repo ARN |
| `/storr/ecr/pipeline-repo-uri` | Pipeline ECR repo URI |
| `/storr/ecr/pipeline-repo-arn` | Pipeline ECR repo ARN |
| `/storr/codeartifact/domain-name` | CodeArtifact domain name |
| `/storr/codeartifact/domain-arn` | CodeArtifact domain ARN |
| `/storr/codeartifact/storr-repo-arn` | Storr CodeArtifact repository ARN |
| `/storr/codeartifact/storr-repo-endpoint` | Storr CodeArtifact npm endpoint |
| `/storr/s3/eve-source-bucket-name` | Eve source S3 bucket name |
| `/storr/s3/eve-source-bucket-arn` | Eve source S3 bucket ARN |

### 10-network (6 params)
| Parameter | Source |
|-----------|--------|
| `/storr/vpc/id` | VPC ID |
| `/storr/subnet/public-a` | Public subnet A |
| `/storr/subnet/public-b` | Public subnet B |
| `/storr/sg/alb` | ALB security group |
| `/storr/sg/fargate` | Fargate security group |
| `/storr/sg/efs` | EFS security group |

### 20-data (11 params)
| Parameter | Source |
|-----------|--------|
| `/storr/cognito/user-pool-id` | Cognito UserPool ID |
| `/storr/cognito/user-pool-arn` | Cognito UserPool ARN |
| `/storr/cognito/client-id` | Cognito UserPoolClient ID |
| `/storr/dynamodb/stores-table` | Stores table name |
| `/storr/dynamodb/stores-table-arn` | Stores table ARN |
| `/storr/dynamodb/oauth-states-table` | OAuth states table name |
| `/storr/dynamodb/oauth-states-table-arn` | OAuth states table ARN |
| `/storr/dynamodb/products-table` | Products table name |
| `/storr/dynamodb/products-table-arn` | Products table ARN |
| `/storr/dynamodb/webhook-events-table` | Webhook events table name |
| `/storr/dynamodb/webhook-events-table-arn` | Webhook events table ARN |

### 30-storage (4 params)
| Parameter | Source |
|-----------|--------|
| `/storr/efs/filesystem-id` | EFS filesystem ID |
| `/storr/efs/filesystem-arn` | EFS filesystem ARN |
| `/storr/s3/product-images-bucket-name` | Product images S3 bucket name |
| `/storr/s3/product-images-bucket-arn` | Product images S3 bucket ARN |

### 35-config (8 params)
| Parameter | Source |
|-----------|--------|
| `/storr/figma-redirect-uri` | Figma OAuth redirect URI |
| `/storr/shopify-redirect-uri` | Shopify OAuth redirect URI |
| `/storr/llm-model` | LLM model name |
| `/storr/llm-provider` | LLM provider |
| `/storr/secrets/figma-arn` | Figma Secrets Manager secret ARN |
| `/storr/secrets/shopify-arn` | Shopify Secrets Manager secret ARN |
| `/storr/secrets/anthropic-arn` | Anthropic Secrets Manager secret ARN |
| `/storr/secrets/runtime-auth-arn` | Runtime auth Secrets Manager secret ARN |

### 40-services (12 params)
| Parameter | Source |
|-----------|--------|
| `/storr/ecs/cluster-name` | ECS cluster name |
| `/storr/ecs/cluster-arn` | ECS cluster ARN |
| `/storr/alb/arn` | ALB ARN |
| `/storr/alb/dns-name` | ALB DNS name |
| `/storr/alb/listener-arn` | HTTPS listener ARN |
| `/storr/alb/full-name` | ALB full name |
| `/storr/ecs/fargate-sg` | Fargate security group |
| `/storr/ecs/subnet-ids` | Comma-separated subnet IDs |
| `/storr/iam/studio-execution-role-arn` | Studio execution role ARN |
| `/storr/iam/studio-task-role-arn` | Studio task role ARN |
| `/storr/ecs/studio-task-def-arn` | Studio task definition ARN |
| `/storr/ecs/pipeline-task-def-arn` | Pipeline task definition ARN |

### 45-api (3 params)
| Parameter | Source |
|-----------|--------|
| `/storr/api/url` | API URL |
| `/storr/api/function-name` | Lambda function name |
| `/storr/api/function-arn` | Lambda function ARN |

### 50-frontend (3 params)
| Parameter | Source |
|-----------|--------|
| `/storr/cloudfront/distribution-id` | CloudFront distribution ID |
| `/storr/cloudfront/domain-name` | CloudFront domain name |
| `/storr/s3/website-bucket-name` | Website S3 bucket name |

### 55-cicd (0 params)
Consumes only  - reads ECR repo URIs/ARNs, S3 bucket names, Lambda function names, and CloudFront distribution ID. No SSM parameters written.

### 60-observability (0 params)
Consumes only  - reads Lambda function name, ALB full name, and DynamoDB table names. No SSM parameters written.

---

## End-to-End Data Flow

```
User (browser)
    |
app.storr.build (CloudFront -> S3)
    |
api.storr.build (API Gateway -> Lambda)
    |
    ├── Cognito (user authentication)
    ├── DynamoDB (stores, products, oauth-states, webhook-events)
    ├── S3 product-images (user-uploaded images)
    |
    └── ECS Fargate (studio/pipeline containers)
            |
            ├── EFS (theme files, build artifacts)
            ├── S3 eve-source (Eve theme framework)
            └── ALB (*.preview.storr.build -> studio containers)
                    |
                User views live preview in browser
```

**OAuth Flow (Figma/Shopify):**
1. User clicks "Connect" in app.storr.build
2. Frontend redirects to API `/oauth/{provider}/authorize`
3. API generates state token, stores in DynamoDB oauth-states table, redirects to provider
4. Provider redirects back to API `/oauth/{provider}/callback` with code
5. API validates state, exchanges code for access token, stores in Secrets Manager, redirects to frontend

**Studio Launch Flow:**
1. User opens studio for a store in app.storr.build
2. Frontend calls API `POST /studios`
3. API Lambda:
   - Creates EFS access point for the store
   - Creates ALB target group and listener rule for `<store-id>.preview.storr.build`
   - Runs ECS task (storr-studio) with EFS mount
   - Returns studio URL to frontend
4. User edits theme in browser at `<store-id>.preview.storr.build`
5. Studio container auto-shuts down after 30 minutes of inactivity

**Theme Build Flow:**
1. User clicks "Deploy" in studio
2. Frontend calls API `POST /builds`
3. API Lambda runs ECS task (storr-pipeline) with EFS mount and eve-source S3 bucket
4. Pipeline container builds Shopify theme, uploads to Shopify via Admin API
5. Task completes, build artifacts remain in EFS for future reference

---

## Teardown

Delete stacks in reverse order (60 to 00) from the CloudFormation console. Custom resources in several stacks handle cleanup of non-CloudFormation-managed resources automatically during deletion.

| Order | Stack | Notes |
|-------|-------|-------|
| 1 | **60-observability** | No special considerations. |
| 2 | **55-cicd** | S3 artifact bucket is auto-emptied by custom resource. Deleting this stops all CI/CD pipelines. |
| 3 | **50-frontend** | S3 website bucket is auto-emptied by custom resource. |
| 4 | **45-api** | Placeholder zip is auto-deleted from S3 by custom resource. |
| 5 | **40-services** | Custom resource automatically stops running ECS tasks, deletes ALB listener rules, and deletes target groups. |
| 6 | **35-config** | Secrets Manager secrets and SSM parameters are deleted with the stack. |
| 7 | **30-storage** | S3 product-images bucket is auto-emptied by custom resource. EFS filesystem has `DeletionPolicy: Retain`. After stack deletion, manually delete the retained EFS filesystem and its access points from the console if desired. |
| 8 | **20-data** | DynamoDB tables and Cognito UserPool have `DeletionPolicy: Retain`. After stack deletion, manually delete these from the console if desired. |
| 9 | **10-network** | No special considerations. |
| 10 | **05-foundation** | ECR repos and all images are deleted. S3 eve-source bucket is auto-emptied by custom resource. CodeArtifact domain and repos are deleted. KMS key is scheduled for deletion (30-day waiting period). |
| 11 | **00-role** | Delete last, and **always with a role override** from the CLI, never the console Delete button: `aws cloudformation delete-stack --stack-name <role stack> --role-arn arn:aws:iam::<account>:role/CloudFormationGitSyncRole`. The stack's stored service role is `StorrGitSyncRole`, which is also one of its resources; in reverse dependency order CloudFormation deletes the role first (detaching the four managed policies), so without the override it loses the credentials mid-delete and every "Retry delete" reassumes the now-deleted role and fails with "role ... is invalid or cannot be assumed." The external `CloudFormationGitSyncRole` is not a stack resource, so it survives the full deletion. If that role lacks `iam:DeletePolicy`, add `--retain-resources StorrCoreServicesPolicy StorrNetworkAndComputePolicy StorrDataAndAppPolicy StorrDeliveryAndObservabilityPolicy` and delete those four policies by hand in IAM afterward (confirm all four `Storr*Policy` are gone before recreating, or the create collides). Deleting this stack while other stacks still use `StorrGitSyncRole` as their service role would strand them. See ADR-0007. |

**Note:** Route 53 records created by stacks (e.g., `*.preview.storr.build`, `api.storr.build`, `app.storr.build`) are deleted with their respective stacks. The `storr.build` hosted zone itself is not created by these templates and is not affected by teardown.
