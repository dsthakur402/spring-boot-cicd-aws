# Part 1: CI/CD Design for CloudEagle Sync-Service

## Overview
This document outlines a simple, effective CI/CD pipeline design for CloudEagle's Spring Boot sync-service, which connects to MongoDB and deploys to AWS environments (QA, Staging, Production).

## 1. Branching Strategy

### Simple Git Flow
We use a straightforward branching strategy that's easy to understand and maintain:

```
main branch (production)
├── develop branch (integration and QA)
├── release/* branches (staging)
└── feature/* branches (development work)
```

### Branch-to-Environment Mapping

| Branch Pattern | Environment | Auto-Deploy | Approval Required |
|----------------|-------------|-------------|-------------------|
| `feature/*`    | QA          | Yes         | No                |
| `develop`      | QA          | Yes         | No                |
| `release/*`    | Staging     | Yes         | No                |
| `main`         | Production  | Yes         | Yes               |

### Safety Rules

1. **Branch Protection**
   - No direct pushes to `main` or `develop` branches
   - Pull requests required with at least 1 reviewer
   - All tests must pass before merge

2. **Production Safety**
   - Manual approval required for production deployments with confirmation dialog
   - Health checks must pass after deployment
   - Simple rollback process available

### Build Versioning

**Build Version Format**: `YYYYMMDD_HHMMSS-{7-char-git-commit}`
- **Example**: `20241223_143000-a1b2c3d`
- **Generated using**: `date +%Y%m%d_%H%M%S` + `git rev-parse --short HEAD`
- **Used for**: Docker image tags, deployment tracking, and notifications
- **Jenkins Display**: Build name shows as `#{BUILD_NUMBER} - {BUILD_TIMESTAMP}`

## 2. Jenkins Pipeline Stages

### Pipeline Flow
Our pipeline consists of simple, easy-to-understand stages:

```
1. Checkout → 2. Build → 3. Test → 4. Docker Build → 5. Push to ECR → 6. Deploy → 7. Health Check
```

### Stage Details

**1. Checkout**
- Get latest code from Git using `checkout scm`
- Set build version using timestamp and git commit (format: `YYYYMMDD_HHMMSS-gitcommit`)
- Update build display name for easy identification

**2. Build**
- Clean and compile using Maven wrapper (`./mvnw clean compile`)
- Package application with tests enabled (`./mvnw package -DskipTests=false`)
- Build outputs JAR file in target/ directory

**3. Test**
- Execute unit tests with Maven wrapper (`./mvnw test`)
- Publish test results from `target/surefire-reports/*.xml`
- Archive built JAR artifacts from `target/*.jar`

**4. Build Docker Image** 
- Build Docker image with BUILD_VERSION tag
- Also tag image as 'latest' for convenience
- Display built images for verification

**5. Push to ECR**
- Authenticate with AWS ECR using `aws ecr get-login-password`
- Push both versioned and latest Docker images
- Target ECR repository: `123456789012.dkr.ecr.us-west-2.amazonaws.com/sync-service`

**6. Deploy**
- Auto-detect environment from branch name:
  - All branches default to 'qa'
  - `release/*` → staging
  - `main` → production
- For production: Manual approval required with confirmation dialog
- Execute ECS deployment with `aws ecs update-service --force-new-deployment`
- Wait for deployment stability using `aws ecs wait services-stable`

**7. Health Check**
- Query load balancer DNS name (`sync-service-lb`)
- Test `/actuator/health` endpoint with curl
- Retry up to 10 times with 30-second intervals between attempts
- Confirm successful deployment before pipeline completion

### Deployment Triggers

**Automatic Deployments:**
- `feature/*` branches → QA environment
- `develop` branch → QA environment  
- `release/*` branches → Staging environment
- `main` branch → Production (with manual approval step)

**Environment Detection Logic:**
```groovy
def environment = 'qa'  // default
if (env.BRANCH_NAME == 'main') {
    environment = 'production'
} else if (env.BRANCH_NAME.startsWith('release/')) {
    environment = 'staging'
}
```

### Rollback Strategy

**Available Rollback Options:**
1. **Automatic ECS Rollback**: ECS service automatically rolls back if health checks fail
2. **Manual Pipeline Re-run**: Trigger pipeline with previous successful commit
3. **Direct ECS Update**: Update service to previous task definition manually
4. **Docker Cleanup**: Pipeline automatically cleans up unused images with `docker system prune -f`

## 3. Configuration Management

### Environment-Specific Configuration

**Spring Boot Profiles**: We use Spring Boot's built-in profile system:

```
src/main/resources/
├── application.yml              # Default settings
├── application-qa.yml           # QA environment settings  
├── application-staging.yml      # Staging environment settings
└── application-prod.yml         # Production environment settings
```

### Configuration Examples

**application.yml (Base Configuration)**
```yaml
spring:
  application:
    name: sync-service
server:
  port: 8080
logging:
  level:
    com.cloudeagle: INFO
```

**application-prod.yml (Production Overrides)**
```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}  # From environment variable
logging:
  level:
    com.cloudeagle: WARN
    root: ERROR
```

### Secrets Management

**Environment Variables**: Simple and secure approach using AWS ECS:

1. **Database Connection**
   - Store MongoDB URI in AWS Secrets Manager
   - ECS task definition injects as environment variable
   - Application reads via `${MONGODB_URI}`

2. **API Keys**
   - Store in AWS Secrets Manager
   - Reference in ECS task definition
   - Never commit secrets to Git

**Example ECS Task Definition:**
```json
{
  "secrets": [
    {
      "name": "MONGODB_URI",
      "valueFrom": "arn:aws:secretsmanager:us-west-2:123456789:secret:cloudeagle/mongodb-uri"
    },
    {
      "name": "JWT_SECRET",  
      "valueFrom": "arn:aws:secretsmanager:us-west-2:123456789:secret:cloudeagle/jwt-secret"
    }
  ]
}

## 4. Deployment Strategy

### Chosen Strategy: Rolling Deployment

**Why Rolling Deployment?**

✅ **Pros**:
- **Simple to understand and implement**
- **Cost effective** - no extra infrastructure needed
- **Gradual rollout** - issues affect fewer users initially  
- **Built-in AWS support** with ECS

❌ **Cons**:
- **Brief downtime** during container replacement
- **Slower rollback** compared to Blue/Green

### Deployment Process

**ECS Rolling Update:**
```
Current: [Container 1] [Container 2] [Container 3]
Step 1:  [Container 1] [Container 2] [New Container 3] ← Deploy new version
Step 2:  [Container 1] [New Container 2] [New Container 3] ← Replace next
Step 3:  [New Container 1] [New Container 2] [New Container 3] ← Complete
```

**Deployment Steps:**
1. **Build & Push** new Docker image to ECR with version tag
2. **Update ECS Service** using AWS CLI:
   ```bash
   aws ecs update-service \
       --cluster cloudeagle-${environment} \
       --service sync-service \
       --force-new-deployment \
       --task-definition sync-service:latest
   ```
3. **Wait for Deployment Stability**:
   ```bash
   aws ecs wait services-stable \
       --cluster cloudeagle-${environment} \
       --services sync-service
   ```
4. **Verify** deployment with health endpoint checks

### Configuration for Zero-Downtime

**ECS Service Settings:**
```json
{
  "deploymentConfiguration": {
    "maximumPercent": 200,        // Allow 2x containers during deployment
    "minimumHealthyPercent": 50   // Keep at least 50% running
  }
}
```

This means:
- **During deployment**: Up to 6 containers can run (3 old + 3 new)
- **Always maintain**: At least 1-2 containers running
- **Health checks**: New containers must pass before old ones stop

### Health Checks

**Pipeline Health Check Implementation:**
- **Load Balancer Discovery**: Query `sync-service-lb` for DNS name
- **Health Endpoint**: `http://{load-balancer-url}/actuator/health`
- **Retry Logic**: Up to 10 attempts with 30-second intervals
- **Verification**: Uses `curl -f` to validate HTTP 200 response
- **Failure Handling**: Pipeline fails if health checks don't pass within 5 minutes

**Health Check Script:**
```bash
LB_URL=$(aws elbv2 describe-load-balancers \
    --names sync-service-lb \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

for i in {1..10}; do
    if curl -f "http://${LB_URL}/actuator/health"; then
        echo "✅ Health check passed!"
        break
    else
        echo "⏳ Health check attempt $i/10..."
        sleep 30
    fi
done
```

### Simple Rollback Process

**If Deployment Fails:**
1. **Automatic**: ECS rolls back if health checks fail
2. **Manual**: Update service to use previous task definition
```bash
aws ecs update-service \
    --cluster cloudeagle-prod \
    --service sync-service \
    --task-definition sync-service:previous
```

## 5. Pipeline Post-Actions

### Cleanup Process
**Always Execute (regardless of pipeline result):**
- Docker system cleanup: `docker system prune -f`
- Removes unused Docker images to save Jenkins agent disk space
- Prevents build agents from running out of storage

### Success Actions
- Console log confirmation message
- Slack notification to `#devops` channel with success status
- Includes deployment version and branch information

### Failure Actions  
- Console log error message
- Slack notification to `#devops` channel with failure alert
- Includes build number and link to Jenkins console output for debugging

## 6. Notifications & Monitoring

### Build Notifications

**Slack Integration:**
- **Channel**: `#devops`
- **Success Notifications** (green color):
  ```
  ✅ Deployment successful for sync-service (develop) - Version: 20241223_143000-a1b2c3d
  ```
- **Failure Notifications** (red color):
  ```
  ❌ Deployment failed for sync-service (develop) - Build: #42. Check: http://jenkins-url/build/42/console
  ```

**Notification Details Include:**
- Application name (sync-service)
- Branch name
- Build version (timestamp-gitcommit format)
- Build number and console URL for failures

### Basic Monitoring
**Health Checks:**
- Application health endpoint (`/actuator/health`)
- Database connectivity check
- ECS service health monitoring

**AWS CloudWatch:**
- Application logs from ECS containers
- Basic metrics (CPU, memory, requests)
- Alerts for high error rates

## 7. Security Best Practices

### Secrets Management
- ✅ **Never commit secrets to Git**
- ✅ **Use AWS Secrets Manager** for sensitive data
- ✅ **Environment variables** for configuration
- ✅ **IAM roles** with minimal permissions

### Docker Security
- Use official base images (e.g., `openjdk:17-jre-slim`)
- Regularly update base images
- Run containers as non-root user

### Access Control  
- Jenkins access via corporate SSO
- AWS resources accessed via IAM roles
- Environment-specific access controls

## 8. Technical Configuration

### Pipeline Environment Variables
```groovy
environment {
    APP_NAME = 'sync-service'
    AWS_REGION = 'us-west-2'
    ECR_REPOSITORY = '123456789012.dkr.ecr.us-west-2.amazonaws.com/sync-service'
    BUILD_VERSION = "${env.BUILD_TIMESTAMP}-${env.GIT_COMMIT?.take(7)}"
}
```

### AWS Resources Referenced
- **ECR Repository**: `123456789012.dkr.ecr.us-west-2.amazonaws.com/sync-service`
- **ECS Clusters**: 
  - `cloudeagle-qa` (QA environment)
  - `cloudeagle-staging` (Staging environment) 
  - `cloudeagle-production` (Production environment)
- **ECS Service**: `sync-service` (consistent across all environments)
- **Load Balancer**: `sync-service-lb` (for health checks)

### Required Jenkins Plugins
- AWS Steps Plugin (for ECS and ECR integration)
- Docker Pipeline Plugin (for Docker operations)
- Pipeline Plugin (for Jenkinsfile support)
- Slack Notification Plugin (for team notifications)

## 9. Getting Started

### Prerequisites
- Jenkins server with Docker support
- AWS CLI configured with appropriate permissions  
- ECR repository created
- ECS cluster and service set up

### Initial Setup Steps
1. **Configure Jenkins**:
   - Install required plugins (AWS, Docker, Pipeline)
   - Set up AWS credentials
   - Configure Slack notifications

2. **Prepare Application**:
   - Add Dockerfile to repository
   - Configure Spring Boot actuator endpoints
   - Set up environment-specific configuration files

3. **Test Pipeline**:
   - Create feature branch
   - Make a small change
   - Verify automatic QA deployment

4. **Production Readiness**:
   - Test staging deployment
   - Verify manual approval process
   - Test rollback procedure

This simple, effective CI/CD design provides reliable deployments for the CloudEagle sync-service while being easy to understand and maintain.