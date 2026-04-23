# CloudEagle Sync-Service - AWS & GCP Architecture Diagram

## High-Level Architecture Overview

<img width="1080" height="748" alt="image" src="https://github.com/user-attachments/assets/d87ad17a-a033-4246-8903-4af13706a4cc" />



## Architecture Components

### 1. **Internet Gateway & DNS**
- **Route 53**: DNS routing to load balancer
- **Internet Gateway**: Provides internet access to VPC

### 2. **Load Balancing Layer**  
- **Application Load Balancer (ALB)**:
  - Distributes traffic across ECS containers
  - Health checks on `/actuator/health`
  - HTTPS termination with SSL certificate
  - Cross-AZ load balancing

### 3. **Compute Layer - ECS (Elastic Container Service)**
- **ECS Service**: Manages sync-service containers
- **ECS Tasks**: Run Docker containers across multiple AZs
- **Auto Scaling**: Scale containers based on CPU/memory usage
- **Service Discovery**: Internal service communication

### 4. **Database Layer**
- **Amazon DocumentDB** (MongoDB-compatible):
  - Multi-AZ deployment for high availability
  - Automatic backups and point-in-time recovery  
  - Cluster endpoint for read/write operations
  - Reader endpoint for read scaling

### 5. **Security & Secrets**
- **AWS Secrets Manager**: 
  - Database credentials
  - API keys and JWT secrets
  - Automatic rotation capabilities

### 6. **Monitoring & Logging**
- **CloudWatch Logs**: Application and system logs
- **CloudWatch Metrics**: Performance monitoring
- **CloudWatch Alarms**: Automated alerting

### 7. **Container Registry**
- **ECR (Elastic Container Registry)**: 
  - Private Docker image storage
  - Image scanning for vulnerabilities
  - Integration with ECS deployments

## Network Architecture

### VPC Configuration
```
CloudEagle VPC (10.0.0.0/16)
│
├── Public Subnets (Web Tier)
│   ├── 10.0.1.0/24 (AZ-1a) - ALB
│   ├── 10.0.2.0/24 (AZ-1b) - ALB  
│   └── 10.0.3.0/24 (AZ-1c) - ALB
│
├── Private Subnets (App Tier)
│   ├── 10.0.11.0/24 (AZ-1a) - ECS Containers
│   ├── 10.0.12.0/24 (AZ-1b) - ECS Containers
│   └── 10.0.13.0/24 (AZ-1c) - ECS Containers
│
└── Database Subnets (Data Tier)
    ├── 10.0.21.0/24 (AZ-1a) - DocumentDB
    ├── 10.0.22.0/24 (AZ-1b) - DocumentDB
    └── 10.0.23.0/24 (AZ-1c) - DocumentDB
```

### Security Groups

**ALB Security Group:**
- Inbound: Port 80/443 from Internet (0.0.0.0/0)
- Outbound: Port 8080 to ECS Security Group

**ECS Security Group:**
- Inbound: Port 8080 from ALB Security Group
- Outbound: Port 27017 to DocumentDB Security Group
- Outbound: Port 443 to Internet (for AWS API calls)

**DocumentDB Security Group:**
- Inbound: Port 27017 from ECS Security Group
- Outbound: None

## Auto Scaling Configuration

### ECS Service Auto Scaling
```yaml
Target Tracking Scaling:
  - Metric: Average CPU Utilization
  - Target: 70%
  - Scale Out: +1 task when CPU > 70% for 2 minutes
  - Scale In: -1 task when CPU < 70% for 5 minutes
  
Min Capacity: 2 tasks
Max Capacity: 10 tasks
```

### Environment-Specific Scaling

| Environment | Min Tasks | Max Tasks | Target CPU |
|-------------|-----------|-----------|------------|
| QA          | 1         | 3         | 80%        |
| Staging     | 2         | 5         | 75%        |
| Production  | 3         | 10        | 70%        |

## High Availability & Disaster Recovery

### Multi-AZ Deployment
- **ECS Tasks**: Distributed across 3 Availability Zones
- **DocumentDB**: Multi-AZ cluster with automatic failover
- **Load Balancer**: Cross-AZ load balancing

### Backup Strategy
- **DocumentDB**: Automated daily backups (7-day retention)
- **Application Code**: Stored in Git repositories
- **Docker Images**: Versioned in ECR with lifecycle policies

### Failover Process
1. **Container Failure**: ECS automatically replaces failed containers
2. **AZ Failure**: ALB routes traffic to healthy AZs automatically  
3. **Database Failure**: DocumentDB fails over to standby instance
4. **Region Failure**: Manual failover to backup region (future enhancement)

## Cost Optimization Features

### Right-Sizing
- **ECS**: Use Fargate Spot instances where appropriate
- **DocumentDB**: Start with smaller instance sizes, scale as needed
- **ALB**: Single ALB serving multiple environments with path-based routing

### Lifecycle Management  
- **ECR**: Automatically delete old Docker images (keep last 10)
- **CloudWatch Logs**: 30-day retention for application logs
- **EBS Snapshots**: Automated cleanup of old backups

### Reserved Capacity
- **DocumentDB**: Use Reserved Instances for production (1-year term)
- **Data Transfer**: Keep traffic within same AZ when possible
