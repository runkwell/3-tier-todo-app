## Project Overview

A full-stack task management application demonstrating production-ready cloud architecture with:

- **3-Tier Architecture** - Presentation, Application, and Data layers
- **Multi-AZ Deployment** - High availability across 2 availability zones
- **Auto-Scaling** - Dynamic instance scaling (2-6 EC2) based on CPU metrics
- **Zero-Downtime Deployments** - Rolling updates with 80% minimum healthy capacity
- **Global CDN** - CloudFront for static and dynamic content delivery
- **Automated CI/CD** - GitLab pipeline from commit to production
- **Infrastructure Monitoring** - CloudWatch dashboards and SNS alerts
- **Security Best Practices** - Private subnets, IAM roles, JWT authentication

## Cloud Architecture
![Architecture](./Architecture/architecture_3_tier_app.png)

## Setup Instructions

### 1. Jenkins CI/CD Variables

Configure in **Settings → CI/CD → Variables**:

| Variable | Example |
|----------|---------|
| `AWS_REGION` | `ap-southeast-1` | 
| `ECR_REPO` | `123456789.dkr.ecr.ap-southeast-2.amazonaws.com` | 
| `ECR_BACKEND` | `${ECR_REPO}/todoapp-backend` | 
| `ECR_FRONTEND` | `${ECR_REPO}/todoapp-frontend` | 
| `LT_VERSION` | `lt-0abc123def456` | 
| `ASG_NAME` | `todoapp-production-asg` | 
| `DEPLOY_DIR` | `/opt/todo-app` | 
| `SQLALCHEMY_DATABASE_URI` | `postgresql://admin:pass@rds:5432/todoapp` |
| `JWT_SECRET_KEY` | `your-secret-key-min-32-chars` |

### 2. IAM Role for EC2 Jenkins

Attach to runner EC2 instance:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ecr:GetAuthorizationToken",
      "ecr:BatchGetImage",
      "ecr:PutImage",
      "ec2:DescribeLaunchTemplates",
      "ec2:DescribeLaunchTemplateVersions",
      "ec2:CreateLaunchTemplateVersion",
      "autoscaling:DescribeAutoScalingGroups",
      "autoscaling:StartInstanceRefresh",
      "autoscaling:DescribeInstanceRefreshes"
    ],
    "Resource": "*"
  }]
}
