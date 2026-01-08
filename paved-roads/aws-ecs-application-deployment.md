# AWS ECS Application Deployment with CodePipeline

## Problem Statement

Teams need a standardized, repeatable way to deploy containerized applications to AWS with automated CI/CD pipelines, proper environment separation, load balancing, and CDN/WAF protection for external-facing services.

## Solution Overview

This paved road provides a complete deployment pipeline for containerized applications running on AWS ECS Fargate. The solution leverages AWS CodePipeline and CodeBuild for CI/CD, separates deployment infrastructure by environment, and integrates with Cloudflare for external-facing applications to provide caching and WAF capabilities.

### Key Components

- **Source Control**: GitHub repository with application code
- **Build Pipeline**: AWS CodePipeline and CodeBuild (per environment)
- **Container Registry**: Amazon ECR (in workloads account)
- **Compute**: ECS Fargate (in workloads account)
- **Load Balancing**: Application Load Balancer with SSL termination
- **CDN/WAF**: Cloudflare (for external applications)

## Architecture Diagram

```mermaid
graph LR
    Dev[Developer] -->|git push| GH[GitHub Repository<br/>buildspec.yml<br/>Dockerfile]
    
    GH -->|Trigger on commit| CP_Dev[CodePipeline<br/>deployments-dev account]
    GH -->|Trigger on commit| CP_Test[CodePipeline<br/>deployments-test account]
    GH -->|Trigger on commit| CP_Prod[CodePipeline<br/>deployments-prod account]
    
    CP_Dev --> CB_Dev[CodeBuild<br/>Build Container]
    CP_Test --> CB_Test[CodeBuild<br/>Build Container]
    CP_Prod --> CB_Prod[CodeBuild<br/>Build Container]
    
    CB_Dev --> ECR[ECR Repository<br/>workloads account]
    CB_Test --> ECR
    CB_Prod --> ECR
    
    ECR --> ECS[ECS Cluster Fargate<br/>workloads account]
    
    ECS --> SVC[ECS Service<br/>Running Tasks ARM64]
    
    SVC --> TG[Target Group]
    TG --> ALB[Application Load Balancer<br/>SSL Termination]
    
    ALB --> CF[Cloudflare<br/>CDN + WAF<br/>SSL Termination]
    
    CF --> User[End Users]
    
    style GH fill:#f9f,stroke:#333,stroke-width:2px
    style ECR fill:#ff9,stroke:#333,stroke-width:2px
    style ECS fill:#9f9,stroke:#333,stroke-width:2px
    style CF fill:#99f,stroke:#333,stroke-width:2px
    style ALB fill:#f99,stroke:#333,stroke-width:2px
```

## Requirements

### Prerequisites

- **GitHub Repository**: Application code with required files
  - `Dockerfile` - Container build instructions
  - `buildspec.yml` - CodeBuild build specification
- **AWS Accounts**: 
  - Deployment accounts: `deployments-dev`, `deployments-test`, `deployments-stage`, `deployments-prod`
  - Workloads account (determined by CloudOps/DevOps based on application requirements)
- **Cloudflare Account**: For external-facing applications
- **Domain Name**: Registered and managed in Cloudflare (if external)

### Repository Structure

```
app-repository/
├── Dockerfile
├── buildspec.yml
├── src/
│   └── (application code)
└── README.md
```

### Branch Strategy

- `dev` branch → triggers deployment to dev environment
- `test` branch → triggers deployment to test environment  
- `stage` branch → triggers deployment to stage environment
- `main` or `prod` branch → triggers deployment to prod environment

## Implementation Guide

### Step 1: Prepare GitHub Repository

1. Ensure repository contains `Dockerfile` and `buildspec.yml`
2. Configure branch protection rules for environment branches
3. Set up appropriate branching strategy (dev/test/stage/prod)

### Step 2: AWS Account Setup

**Per Environment (in deployments-{env} accounts):**

1. Create CodePipeline pipeline
   - Source: GitHub repository (specific branch)
   - Build: CodeBuild project
   - Deploy: ECS service update

2. Create CodeBuild project
   - Environment: AWS managed image for Docker builds
   - Buildspec: Use `buildspec.yml` from repository
   - Permissions: Push to ECR in workloads account

**In Workloads Account:**

3. Create ECR repository
   - Configure image scanning
   - Set lifecycle policies for image retention
   - Grant push permissions to deployment account CodeBuild roles

4. Create ECS Cluster
   - Fargate capacity provider
   - Container Insights enabled (recommended)

5. Create ECS Task Definition
   - **Architecture**: ARM64 (preferred, unless application incompatible)
   - CPU and memory allocations based on application needs
   - Image: Reference ECR repository
   - Logging: CloudWatch Logs

6. Create ECS Service
   - Launch type: Fargate
   - Desired count: Based on environment and scaling needs
   - Load balancer: Attach to target group
   - Deployment configuration: Rolling updates

7. Create Application Load Balancer
   - Scheme: Internet-facing or internal
   - Target group: Forward to ECS service tasks
   - Health checks: Configure appropriate path and thresholds
   - Listener: HTTPS (port 443) with SSL certificate

8. Configure SSL/TLS
   - Request ACM certificate for domain
   - Attach to ALB HTTPS listener
   - Configure security policy (TLS 1.2+)

### Step 3: Cloudflare Configuration (External Applications)

1. Add DNS record pointing to ALB
   - Type: CNAME
   - Name: `{environment}-{app-name}`
   - Target: ALB DNS name
   - Proxy status: Enabled (orange cloud)

2. Configure SSL/TLS
   - Mode: Full (strict) or Full
   - Edge certificates: Automatic

3. Configure Caching
   - **Default**: Enabled (unless engineering team requests otherwise)
   - Cache rules based on application requirements
   - Respect origin headers as appropriate

4. Enable WAF
   - Apply managed rulesets
   - Configure custom rules as needed
   - Set up rate limiting if required

## Configuration

### URL Pattern

External applications follow the pattern:
```
{environment}-{app-name}.{domain}.{com|io}
```

Examples:
- `dev-api.example.com`
- `test-webapp.company.io`
- `prod-service.example.com`

### Environment Variables

Set in ECS Task Definition:
- `ENVIRONMENT` - dev/test/stage/prod
- `AWS_REGION` - AWS region
- Application-specific variables

### Sensible Defaults

**ECS Service:**
- Minimum healthy percent: 100%
- Maximum percent: 200%
- Health check grace period: 60 seconds
- Deployment circuit breaker: Enabled

**Task Definition:**
- CPU: 256 (0.25 vCPU) - adjust based on needs
- Memory: 512 MB - adjust based on needs
- Architecture: ARM64 (use AMD64 if incompatible)

**ALB:**
- Idle timeout: 60 seconds
- Deletion protection: Enabled (for prod)
- Access logs: Enabled to S3

**Target Group:**
- Protocol: HTTP/HTTPS
- Health check interval: 30 seconds
- Healthy threshold: 2
- Unhealthy threshold: 3
- Timeout: 5 seconds

## Operations

### Deployment Process

1. Developer commits code to environment branch (e.g., `dev`)
2. GitHub webhook triggers CodePipeline in corresponding deployment account
3. CodeBuild executes `buildspec.yml`:
   - Builds Docker image
   - Tags image with commit SHA and environment
   - Pushes to ECR in workloads account
4. CodePipeline triggers ECS service update
5. ECS performs rolling deployment:
   - Launches new tasks with updated image
   - Waits for health checks to pass
   - Drains and stops old tasks
6. New version is live

### Monitoring

- **ECS Service**: CloudWatch metrics (CPU, memory, task count)
- **ALB**: Target health, request count, response times
- **CodePipeline**: Build success/failure notifications
- **Cloudflare**: Analytics dashboard for traffic and threats

### Scaling

**Manual Scaling:**
- Update ECS service desired count

**Auto Scaling (recommended for prod):**
- Target tracking: CPU/memory utilization
- Step scaling: Based on CloudWatch alarms
- Scheduled scaling: For predictable traffic patterns

### SSL Certificate Management

- ACM certificates auto-renew
- Cloudflare certificates auto-renew
- Monitor certificate expiration via CloudWatch

## Troubleshooting

### Build Failures

**Symptom**: CodeBuild fails during image build

**Common Causes**:
- Invalid Dockerfile syntax
- Missing dependencies in buildspec.yml
- ECR push permission issues

**Resolution**:
1. Check CodeBuild logs in CloudWatch
2. Verify IAM permissions for CodeBuild role
3. Test Dockerfile locally
4. Validate buildspec.yml syntax

### Deployment Failures

**Symptom**: ECS service fails to stabilize

**Common Causes**:
- Task fails health checks
- Insufficient resources (CPU/memory)
- Container crashes on startup
- Image pull errors

**Resolution**:
1. Check ECS task logs in CloudWatch
2. Verify task definition resource allocations
3. Review application startup logs
4. Confirm ECR repository permissions
5. Check ALB target group health status

### Connection Issues

**Symptom**: Users cannot reach application

**Common Causes**:
- Cloudflare proxy issues
- DNS misconfiguration
- ALB security group restrictions
- Target group health check failures

**Resolution**:
1. Verify Cloudflare DNS settings (orange cloud enabled)
2. Check ALB security groups allow traffic from Cloudflare IPs
3. Confirm target group shows healthy targets
4. Review ALB access logs
5. Test direct ALB endpoint (bypass Cloudflare)

### Architecture Compatibility

**Symptom**: Application won't run on ARM64

**Resolution**:
1. Update task definition to use AMD64 architecture
2. Rebuild container image for AMD64 platform
3. Update buildspec.yml if multi-arch builds needed:
   ```yaml
   docker buildx build --platform linux/amd64 -t ...
   ```

## References

### Related Documentation
- [AWS ECS Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/)
- [AWS CodePipeline Documentation](https://docs.aws.amazon.com/codepipeline/)
- [Cloudflare SSL/TLS Documentation](https://developers.cloudflare.com/ssl/)

### Internal Resources
- CloudOps team contacts for workloads account assignment
- DevOps team for pipeline troubleshooting
- Security team for WAF rule customization

### Sample Files

**Sample buildspec.yml:**
```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $ECR_REPOSITORY:$IMAGE_TAG .
      - docker tag $ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
      - docker tag $ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:$ENVIRONMENT
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
      - docker push $ECR_REGISTRY/$ECR_REPOSITORY:$ENVIRONMENT
      - echo Writing image definitions file...
      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files: imagedefinitions.json
```

---

*Last updated: January 2026*
