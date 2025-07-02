# AWS ECS Fargate + CloudFront + Cloudflare Deployment Guide

## Overview

This guide covers the complete deployment of the Ploy Universal Multi-Chain Loyalty Platform using:
- **AWS ECS Fargate** for containerized application hosting
- **AWS CloudFront** for CDN and static asset delivery
- **Cloudflare** for DNS, security, and additional CDN layers
- **AWS RDS** for PostgreSQL database
- **AWS ElastiCache** for Redis caching
- **AWS Application Load Balancer** for traffic distribution

## Architecture Overview

```mermaid
graph TD
    subgraph "Cloudflare"
        A[DNS Management] --> B[DDoS Protection]
        B --> C[Web Application Firewall]
        C --> D[SSL/TLS Termination]
    end
    
    subgraph "AWS CloudFront"
        E[Global CDN] --> F[Static Assets]
        E --> G[API Caching]
        F --> H[S3 Static Hosting]
    end
    
    subgraph "AWS Load Balancer"
        I[Application Load Balancer] --> J[SSL Certificate]
        I --> K[Health Checks]
    end
    
    subgraph "AWS ECS Fargate"
        L[API Service] --> M[Web Service]
        L --> N[Worker Service]
        M --> O[Admin Dashboard]
        N --> P[Background Jobs]
    end
    
    subgraph "AWS Data Layer"
        Q[RDS PostgreSQL] --> R[Multi-AZ Setup]
        S[ElastiCache Redis] --> T[Cluster Mode]
        U[S3 Buckets] --> V[NFT Metadata]
    end
    
    D --> E
    E --> I
    I --> L
    L --> Q
    L --> S
    M --> U
```

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Infrastructure Setup](#infrastructure-setup)
3. [Container Configuration](#container-configuration)
4. [ECS Fargate Setup](#ecs-fargate-setup)
5. [CloudFront Configuration](#cloudfront-configuration)
6. [Cloudflare Setup](#cloudflare-setup)
7. [CI/CD Pipeline](#cicd-pipeline)
8. [Monitoring & Logging](#monitoring--logging)
9. [Security Configuration](#security-configuration)
10. [Scaling & Performance](#scaling--performance)

---

## Prerequisites

### Required Tools
```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Terraform
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Verify installations
aws --version
terraform --version
docker --version
```

### AWS Account Setup
```bash
# Configure AWS CLI
aws configure
# AWS Access Key ID: [Your Access Key]
# AWS Secret Access Key: [Your Secret Key]
# Default region name: us-east-1
# Default output format: json

# Create IAM role for ECS tasks
aws iam create-role --role-name PloyECSTaskRole --assume-role-policy-document file://ecs-task-role-policy.json
```

### Cloudflare Account
- Domain registered and managed by Cloudflare
- Cloudflare API token with zone edit permissions
- SSL/TLS encryption mode set to "Full (strict)"

---

## Infrastructure Setup

### Terraform Configuration

Create the main infrastructure configuration:

```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
  
  backend "s3" {
    bucket = "ploy-terraform-state"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "Ploy"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}

# Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "domain_name" {
  description = "Primary domain name"
  type        = string
  default     = "ploy.app"
}

variable "cloudflare_zone_id" {
  description = "Cloudflare Zone ID"
  type        = string
}

variable "cloudflare_api_token" {
  description = "Cloudflare API Token"
  type        = string
  sensitive   = true
}
```

### VPC and Networking

```hcl
# terraform/networking.tf
# Create VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "ploy-${var.environment}-vpc"
  }
}

# Create Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "ploy-${var.environment}-igw"
  }
}

# Create public subnets
resource "aws_subnet" "public" {
  count = 2

  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "ploy-${var.environment}-public-${count.index + 1}"
    Type = "Public"
  }
}

# Create private subnets
resource "aws_subnet" "private" {
  count = 2

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "ploy-${var.environment}-private-${count.index + 1}"
    Type = "Private"
  }
}

# Create NAT Gateways
resource "aws_eip" "nat" {
  count = 2

  domain = "vpc"
  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "ploy-${var.environment}-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count = 2

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "ploy-${var.environment}-nat-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "ploy-${var.environment}-public-rt"
  }
}

resource "aws_route_table" "private" {
  count = 2

  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "ploy-${var.environment}-private-rt-${count.index + 1}"
  }
}

# Associate route tables
resource "aws_route_table_association" "public" {
  count = 2

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = 2

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

### Security Groups

```hcl
# terraform/security.tf
# ALB Security Group
resource "aws_security_group" "alb" {
  name_prefix = "ploy-${var.environment}-alb-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ploy-${var.environment}-alb-sg"
  }
}

# ECS Security Group
resource "aws_security_group" "ecs" {
  name_prefix = "ploy-${var.environment}-ecs-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  ingress {
    from_port       = 3001
    to_port         = 3001
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ploy-${var.environment}-ecs-sg"
  }
}

# RDS Security Group
resource "aws_security_group" "rds" {
  name_prefix = "ploy-${var.environment}-rds-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }

  tags = {
    Name = "ploy-${var.environment}-rds-sg"
  }
}

# ElastiCache Security Group
resource "aws_security_group" "elasticache" {
  name_prefix = "ploy-${var.environment}-redis-"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }

  tags = {
    Name = "ploy-${var.environment}-redis-sg"
  }
}
```

### Database Configuration

```hcl
# terraform/database.tf
# RDS Subnet Group
resource "aws_db_subnet_group" "main" {
  name       = "ploy-${var.environment}-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id

  tags = {
    Name = "ploy-${var.environment}-db-subnet-group"
  }
}

# RDS PostgreSQL Instance
resource "aws_db_instance" "main" {
  identifier = "ploy-${var.environment}-db"

  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.medium"

  allocated_storage     = 100
  max_allocated_storage = 1000
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "ploy"
  username = "ploy_admin"
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  multi_az               = true
  publicly_accessible    = false
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "ploy-${var.environment}-final-snapshot"

  performance_insights_enabled = true
  monitoring_interval         = 60
  monitoring_role_arn         = aws_iam_role.rds_monitoring.arn

  tags = {
    Name = "ploy-${var.environment}-db"
  }
}

# ElastiCache Subnet Group
resource "aws_elasticache_subnet_group" "main" {
  name       = "ploy-${var.environment}-redis-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}

# ElastiCache Redis Cluster
resource "aws_elasticache_replication_group" "main" {
  replication_group_id       = "ploy-${var.environment}-redis"
  description                = "Redis cluster for Ploy ${var.environment}"

  node_type               = "cache.t3.medium"
  port                    = 6379
  parameter_group_name    = "default.redis7"
  num_cache_clusters      = 2

  subnet_group_name       = aws_elasticache_subnet_group.main.name
  security_group_ids      = [aws_security_group.elasticache.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token

  automatic_failover_enabled = true
  multi_az_enabled          = true

  tags = {
    Name = "ploy-${var.environment}-redis"
  }
}

# Variables for database
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

variable "redis_auth_token" {
  description = "Redis auth token"
  type        = string
  sensitive   = true
}
```

---

## Container Configuration

### Dockerfile for API Service

```dockerfile
# Dockerfile.api
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY packages/backend/package*.json ./packages/backend/

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY packages/backend/ ./packages/backend/
COPY shared/ ./shared/

# Build the application
RUN npm run build

# Production image
FROM node:18-alpine AS runtime

WORKDIR /app

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S ploy -u 1001

# Copy built application
COPY --from=builder --chown=ploy:nodejs /app/packages/backend/dist ./
COPY --from=builder --chown=ploy:nodejs /app/node_modules ./node_modules

# Set user
USER ploy

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# Start application
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "main.js"]
```

### Dockerfile for Web Service

```dockerfile
# Dockerfile.web
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY packages/frontend/package*.json ./packages/frontend/

# Install dependencies
RUN npm ci

# Copy source code
COPY packages/frontend/ ./packages/frontend/
COPY shared/ ./shared/

# Build the application
ENV NODE_ENV=production
RUN npm run build

# Production image with nginx
FROM nginx:alpine AS runtime

# Copy nginx configuration
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

# Copy built application
COPY --from=builder /app/packages/frontend/dist /usr/share/nginx/html

# Add nginx user
RUN addgroup -g 1001 -S nginx
RUN adduser -S ploy -u 1001 -G nginx

# Set permissions
RUN chown -R ploy:nginx /usr/share/nginx/html
RUN chown -R ploy:nginx /var/cache/nginx

# Expose port
EXPOSE 3001

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3001/health || exit 1

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://ploy_user:ploy_pass@postgres:5432/ploy_dev
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=dev_jwt_secret
    depends_on:
      - postgres
      - redis
    volumes:
      - ./packages/backend:/app/packages/backend
      - /app/node_modules

  web:
    build:
      context: .
      dockerfile: Dockerfile.web
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=development
      - NEXT_PUBLIC_API_URL=http://localhost:3000
    volumes:
      - ./packages/frontend:/app/packages/frontend
      - /app/node_modules

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=ploy_dev
      - POSTGRES_USER=ploy_user
      - POSTGRES_PASSWORD=ploy_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## ECS Fargate Setup

### ECS Cluster Configuration

```hcl
# terraform/ecs.tf
# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "ploy-${var.environment}"

  configuration {
    execute_command_configuration {
      logging = "OVERRIDE"
      log_configuration {
        cloud_watch_log_group_name = aws_cloudwatch_log_group.ecs.name
      }
    }
  }

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name = "ploy-${var.environment}-cluster"
  }
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "ecs" {
  name              = "/ecs/ploy-${var.environment}"
  retention_in_days = 7

  tags = {
    Name = "ploy-${var.environment}-logs"
  }
}

# ECR Repositories
resource "aws_ecr_repository" "api" {
  name = "ploy/api"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Name = "ploy-api-repo"
  }
}

resource "aws_ecr_repository" "web" {
  name = "ploy/web"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Name = "ploy-web-repo"
  }
}

# ECR Lifecycle Policies
resource "aws_ecr_lifecycle_policy" "api" {
  repository = aws_ecr_repository.api.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}

resource "aws_ecr_lifecycle_policy" "web" {
  repository = aws_ecr_repository.web.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

### ECS Task Definitions

```hcl
# terraform/ecs-tasks.tf
# IAM Role for ECS Tasks
resource "aws_iam_role" "ecs_task_execution" {
  name = "ploy-${var.environment}-ecs-task-execution"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# IAM Role for ECS Tasks
resource "aws_iam_role" "ecs_task" {
  name = "ploy-${var.environment}-ecs-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

# Task Definition for API Service
resource "aws_ecs_task_definition" "api" {
  family                   = "ploy-${var.environment}-api"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 1024
  memory                   = 2048
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn           = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "api"
      image = "${aws_ecr_repository.api.repository_url}:latest"

      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        },
        {
          name  = "DATABASE_URL"
          value = "postgresql://${aws_db_instance.main.username}:${var.db_password}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
        },
        {
          name  = "REDIS_URL"
          value = "redis://${aws_elasticache_replication_group.main.primary_endpoint_address}:6379"
        }
      ]

      secrets = [
        {
          name      = "JWT_SECRET"
          valueFrom = aws_secretsmanager_secret.jwt.arn
        },
        {
          name      = "REDIS_AUTH_TOKEN"
          valueFrom = aws_secretsmanager_secret.redis_auth.arn
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.ecs.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "api"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }

      essential = true
    }
  ])

  tags = {
    Name = "ploy-${var.environment}-api-task"
  }
}

# Task Definition for Web Service
resource "aws_ecs_task_definition" "web" {
  family                   = "ploy-${var.environment}-web"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn           = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "web"
      image = "${aws_ecr_repository.web.repository_url}:latest"

      portMappings = [
        {
          containerPort = 3001
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        },
        {
          name  = "NEXT_PUBLIC_API_URL"
          value = "https://api.${var.domain_name}"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.ecs.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "web"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:3001/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }

      essential = true
    }
  ])

  tags = {
    Name = "ploy-${var.environment}-web-task"
  }
}
```

### ECS Services

```hcl
# terraform/ecs-services.tf
# Application Load Balancer
resource "aws_lb" "main" {
  name               = "ploy-${var.environment}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true

  tags = {
    Name = "ploy-${var.environment}-alb"
  }
}

# Target Groups
resource "aws_lb_target_group" "api" {
  name        = "ploy-${var.environment}-api-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }

  tags = {
    Name = "ploy-${var.environment}-api-tg"
  }
}

resource "aws_lb_target_group" "web" {
  name        = "ploy-${var.environment}-web-tg"
  port        = 3001
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }

  tags = {
    Name = "ploy-${var.environment}-web-tg"
  }
}

# Load Balancer Listeners
resource "aws_lb_listener" "main" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.main.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    host_header {
      values = ["api.${var.domain_name}"]
    }
  }
}

# HTTP to HTTPS redirect
resource "aws_lb_listener" "redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# ECS Services
resource "aws_ecs_service" "api" {
  name            = "ploy-${var.environment}-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    security_groups  = [aws_security_group.ecs.id]
    subnets          = aws_subnet.private[*].id
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api"
    container_port   = 3000
  }

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 50
  }

  depends_on = [aws_lb_listener.main]

  tags = {
    Name = "ploy-${var.environment}-api-service"
  }
}

resource "aws_ecs_service" "web" {
  name            = "ploy-${var.environment}-web"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.web.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    security_groups  = [aws_security_group.ecs.id]
    subnets          = aws_subnet.private[*].id
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.web.arn
    container_name   = "web"
    container_port   = 3001
  }

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 50
  }

  depends_on = [aws_lb_listener.main]

  tags = {
    Name = "ploy-${var.environment}-web-service"
  }
}
```

---

## CloudFront Configuration

### S3 Bucket for Static Assets

```hcl
# terraform/s3.tf
# S3 Bucket for static assets
resource "aws_s3_bucket" "static_assets" {
  bucket = "ploy-${var.environment}-static-assets"

  tags = {
    Name = "ploy-${var.environment}-static-assets"
  }
}

resource "aws_s3_bucket_versioning" "static_assets" {
  bucket = aws_s3_bucket.static_assets.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_encryption" "static_assets" {
  bucket = aws_s3_bucket.static_assets.id

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_s3_bucket_public_access_block" "static_assets" {
  bucket = aws_s3_bucket.static_assets.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# S3 Bucket for NFT metadata
resource "aws_s3_bucket" "nft_metadata" {
  bucket = "ploy-${var.environment}-nft-metadata"

  tags = {
    Name = "ploy-${var.environment}-nft-metadata"
  }
}

resource "aws_s3_bucket_versioning" "nft_metadata" {
  bucket = aws_s3_bucket.nft_metadata.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_encryption" "nft_metadata" {
  bucket = aws_s3_bucket.nft_metadata.id

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_s3_bucket_cors_configuration" "nft_metadata" {
  bucket = aws_s3_bucket.nft_metadata.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "HEAD"]
    allowed_origins = ["*"]
    max_age_seconds = 3000
  }
}
```

### CloudFront Distribution

```hcl
# terraform/cloudfront.tf
# Origin Access Control for CloudFront
resource "aws_cloudfront_origin_access_control" "main" {
  name                              = "ploy-${var.environment}-oac"
  description                       = "OAC for Ploy static assets"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  price_class         = "PriceClass_All"

  aliases = [
    var.domain_name,
    "www.${var.domain_name}",
    "api.${var.domain_name}",
    "cdn.${var.domain_name}"
  ]

  # S3 Origin for static assets
  origin {
    domain_name              = aws_s3_bucket.static_assets.bucket_regional_domain_name
    origin_id                = "S3-${aws_s3_bucket.static_assets.bucket}"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }

  # ALB Origin for API
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "ALB-API"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Default cache behavior for static assets
  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${aws_s3_bucket.static_assets.bucket}"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400

    compress = true
  }

  # Cache behavior for API
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD", "OPTIONS"]
    target_origin_id = "ALB-API"

    forwarded_values {
      query_string = true
      headers      = ["Authorization", "CloudFront-Forwarded-Proto", "Host"]
      
      cookies {
        forward = "all"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 0
    max_ttl                = 0

    compress = true
  }

  # Cache behavior for NFT metadata
  ordered_cache_behavior {
    path_pattern     = "/nft/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${aws_s3_bucket.static_assets.bucket}"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 86400
    default_ttl            = 86400
    max_ttl                = 31536000

    compress = true
  }

  # SSL Certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # Geographic restrictions
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  # Custom error pages
  custom_error_response {
    error_code         = 404
    response_code      = 404
    response_page_path = "/404.html"
  }

  custom_error_response {
    error_code         = 403
    response_code      = 403
    response_page_path = "/403.html"
  }

  # Web Application Firewall
  web_acl_id = aws_wafv2_web_acl.main.arn

  tags = {
    Name = "ploy-${var.environment}-cloudfront"
  }
}

# S3 bucket policy for CloudFront
resource "aws_s3_bucket_policy" "static_assets" {
  bucket = aws_s3_bucket.static_assets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudFrontServicePrincipal"
        Effect = "Allow"
        Principal = {
          Service = "cloudfront.amazonaws.com"
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.static_assets.arn}/*"
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = aws_cloudfront_distribution.main.arn
          }
        }
      }
    ]
  })
}
```

### WAF Configuration

```hcl
# terraform/waf.tf
# WAF Web ACL
resource "aws_wafv2_web_acl" "main" {
  name  = "ploy-${var.environment}-waf"
  scope = "CLOUDFRONT"

  default_action {
    allow {}
  }

  # Rate limiting rule
  rule {
    name     = "RateLimitRule"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }

  # AWS Managed Core Rule Set
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 10

    action {
      allow {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesCommonRuleSetMetric"
      sampled_requests_enabled   = true
    }
  }

  # AWS Managed Known Bad Inputs Rule Set
  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 20

    action {
      allow {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesKnownBadInputsRuleSetMetric"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ployWAF"
    sampled_requests_enabled   = true
  }

  tags = {
    Name = "ploy-${var.environment}-waf"
  }
}

# CloudWatch Log Group for WAF
resource "aws_cloudwatch_log_group" "waf" {
  name              = "/aws/wafv2/ploy-${var.environment}"
  retention_in_days = 7

  tags = {
    Name = "ploy-${var.environment}-waf-logs"
  }
}

# WAF Logging Configuration
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  resource_arn            = aws_wafv2_web_acl.main.arn
  log_destination_configs = [aws_cloudwatch_log_group.waf.arn]
}
```

---

## Cloudflare Setup

### SSL/TLS Configuration

```hcl
# terraform/cloudflare.tf
# Cloudflare DNS Records
resource "cloudflare_record" "main" {
  zone_id = var.cloudflare_zone_id
  name    = var.domain_name
  value   = aws_cloudfront_distribution.main.domain_name
  type    = "CNAME"
  proxied = true

  comment = "Main domain pointing to CloudFront"
}

resource "cloudflare_record" "www" {
  zone_id = var.cloudflare_zone_id
  name    = "www"
  value   = aws_cloudfront_distribution.main.domain_name
  type    = "CNAME"
  proxied = true

  comment = "WWW subdomain pointing to CloudFront"
}

resource "cloudflare_record" "api" {
  zone_id = var.cloudflare_zone_id
  name    = "api"
  value   = aws_cloudfront_distribution.main.domain_name
  type    = "CNAME"
  proxied = true

  comment = "API subdomain pointing to CloudFront"
}

resource "cloudflare_record" "cdn" {
  zone_id = var.cloudflare_zone_id
  name    = "cdn"
  value   = aws_cloudfront_distribution.main.domain_name
  type    = "CNAME"
  proxied = true

  comment = "CDN subdomain for static assets"
}

# Cloudflare Zone Settings
resource "cloudflare_zone_settings_override" "main" {
  zone_id = var.cloudflare_zone_id

  settings {
    # SSL/TLS
    ssl = "full"
    always_use_https = "on"
    tls_1_3 = "on"
    automatic_https_rewrites = "on"
    
    # Performance
    brotli = "on"
    minify {
      css  = "on"
      js   = "on"
      html = "on"
    }
    
    # Security
    security_level = "medium"
    challenge_ttl = 1800
    browser_check = "on"
    hotlink_protection = "on"
    
    # Caching
    browser_cache_ttl = 14400
    always_online = "on"
    development_mode = "off"
  }
}

# Page Rules for optimization
resource "cloudflare_page_rule" "api_bypass_cache" {
  zone_id  = var.cloudflare_zone_id
  target   = "api.${var.domain_name}/*"
  priority = 1

  actions {
    cache_level = "bypass"
    ssl = "full"
  }
}

resource "cloudflare_page_rule" "static_cache" {
  zone_id  = var.cloudflare_zone_id
  target   = "cdn.${var.domain_name}/*"
  priority = 2

  actions {
    cache_level = "cache_everything"
    edge_cache_ttl = 86400
    browser_cache_ttl = 86400
  }
}

# Cloudflare Workers for additional logic
resource "cloudflare_worker_script" "security_headers" {
  zone_id = var.cloudflare_zone_id
  name    = "ploy-security-headers"
  content = file("${path.module}/workers/security-headers.js")
}

resource "cloudflare_worker_route" "security_headers" {
  zone_id     = var.cloudflare_zone_id
  pattern     = "${var.domain_name}/*"
  script_name = cloudflare_worker_script.security_headers.name
}
```

### Cloudflare Worker for Security Headers

```javascript
// workers/security-headers.js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const response = await fetch(request)
  const newResponse = new Response(response.body, response)

  // Security headers
  newResponse.headers.set('X-Frame-Options', 'DENY')
  newResponse.headers.set('X-Content-Type-Options', 'nosniff')
  newResponse.headers.set('X-XSS-Protection', '1; mode=block')
  newResponse.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')
  newResponse.headers.set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()')
  
  // Content Security Policy
  const csp = [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.jsdelivr.net",
    "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
    "font-src 'self' https://fonts.gstatic.com",
    "img-src 'self' data: https:",
    "connect-src 'self' https://api.ploy.app wss://api.ploy.app",
    "frame-ancestors 'none'",
    "base-uri 'self'",
    "form-action 'self'"
  ].join('; ')
  
  newResponse.headers.set('Content-Security-Policy', csp)
  
  // HSTS
  newResponse.headers.set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains; preload')

  return newResponse
}
```

---

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm run test
      
      - name: Run lint
        run: npm run lint
      
      - name: Type check
        run: npm run type-check

  build:
    name: Build and Push Images
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build, tag, and push API image
        env:
          ECR_REPOSITORY: ploy/api
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -f Dockerfile.api -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
      
      - name: Build, tag, and push Web image
        env:
          ECR_REPOSITORY: ploy/web
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -f Dockerfile.web -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

  deploy:
    name: Deploy to ECS
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy API to ECS
        run: |
          aws ecs update-service \
            --cluster ploy-production \
            --service ploy-production-api \
            --force-new-deployment
      
      - name: Deploy Web to ECS
        run: |
          aws ecs update-service \
            --cluster ploy-production \
            --service ploy-production-web \
            --force-new-deployment
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster ploy-production \
            --services ploy-production-api ploy-production-web

  invalidate-cache:
    name: Invalidate CloudFront Cache
    runs-on: ubuntu-latest
    needs: deploy
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

### Deployment Scripts

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

# Configuration
ENVIRONMENT=${1:-production}
AWS_REGION=${AWS_REGION:-us-east-1}
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

echo "🚀 Deploying Ploy to ${ENVIRONMENT} environment"

# Build and push images
echo "📦 Building Docker images..."
docker build -f Dockerfile.api -t ploy/api:latest .
docker build -f Dockerfile.web -t ploy/web:latest .

echo "🏷️  Tagging images..."
docker tag ploy/api:latest "${ECR_REGISTRY}/ploy/api:latest"
docker tag ploy/web:latest "${ECR_REGISTRY}/ploy/web:latest"

echo "⬆️  Pushing to ECR..."
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
docker push "${ECR_REGISTRY}/ploy/api:latest"
docker push "${ECR_REGISTRY}/ploy/web:latest"

# Deploy infrastructure
echo "🏗️  Deploying infrastructure..."
cd terraform
terraform init
terraform plan -var-file="${ENVIRONMENT}.tfvars"
terraform apply -var-file="${ENVIRONMENT}.tfvars" -auto-approve

# Update ECS services
echo "🔄 Updating ECS services..."
aws ecs update-service \
  --cluster "ploy-${ENVIRONMENT}" \
  --service "ploy-${ENVIRONMENT}-api" \
  --force-new-deployment

aws ecs update-service \
  --cluster "ploy-${ENVIRONMENT}" \
  --service "ploy-${ENVIRONMENT}-web" \
  --force-new-deployment

# Wait for deployment
echo "⏳ Waiting for deployment to complete..."
aws ecs wait services-stable \
  --cluster "ploy-${ENVIRONMENT}" \
  --services "ploy-${ENVIRONMENT}-api" "ploy-${ENVIRONMENT}-web"

# Invalidate CloudFront cache
echo "🗑️  Invalidating CloudFront cache..."
DISTRIBUTION_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='ploy-${ENVIRONMENT}'].Id" --output text)
aws cloudfront create-invalidation \
  --distribution-id ${DISTRIBUTION_ID} \
  --paths "/*"

echo "✅ Deployment completed successfully!"
```

---

## Monitoring & Logging

### CloudWatch Configuration

```hcl
# terraform/monitoring.tf
# CloudWatch Dashboard
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "Ploy-${var.environment}"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6

        properties = {
          metrics = [
            ["AWS/ECS", "CPUUtilization", "ServiceName", "ploy-${var.environment}-api", "ClusterName", "ploy-${var.environment}"],
            [".", "MemoryUtilization", ".", ".", ".", "."],
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", aws_lb.main.arn_suffix],
            [".", "RequestCount", ".", "."]
          ]
          view    = "timeSeries"
          stacked = false
          region  = var.aws_region
          title   = "ECS and ALB Metrics"
          period  = 300
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6

        properties = {
          metrics = [
            ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", aws_db_instance.main.id],
            [".", "DatabaseConnections", ".", "."],
            ["AWS/ElastiCache", "CPUUtilization", "CacheClusterId", aws_elasticache_replication_group.main.replication_group_id],
            [".", "CurrConnections", ".", "."]
          ]
          view    = "timeSeries"
          stacked = false
          region  = var.aws_region
          title   = "Database and Cache Metrics"
          period  = 300
        }
      }
    ]
  })
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "ploy-${var.environment}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ECS CPU utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ServiceName = aws_ecs_service.api.name
    ClusterName = aws_ecs_cluster.main.name
  }
}

resource "aws_cloudwatch_metric_alarm" "high_memory" {
  alarm_name          = "ploy-${var.environment}-high-memory"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "MemoryUtilization"
  namespace           = "AWS/ECS"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ECS memory utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ServiceName = aws_ecs_service.api.name
    ClusterName = aws_ecs_cluster.main.name
  }
}

resource "aws_cloudwatch_metric_alarm" "database_cpu" {
  alarm_name          = "ploy-${var.environment}-database-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = "300"
  statistic           = "Average"
  threshold           = "75"
  alarm_description   = "This metric monitors RDS CPU utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.id
  }
}

# SNS Topic for alerts
resource "aws_sns_topic" "alerts" {
  name = "ploy-${var.environment}-alerts"

  tags = {
    Name = "ploy-${var.environment}-alerts"
  }
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

variable "alert_email" {
  description = "Email address for alerts"
  type        = string
}
```

### Application Logging

```typescript
// src/logging/logger.ts
import { createLogger, format, transports } from 'winston';
import CloudWatchTransport from 'winston-cloudwatch';

const logger = createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: format.combine(
    format.timestamp(),
    format.errors({ stack: true }),
    format.json()
  ),
  defaultMeta: {
    service: 'ploy-api',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION
  },
  transports: [
    new transports.Console({
      format: format.combine(
        format.colorize(),
        format.simple()
      )
    })
  ]
});

// Add CloudWatch transport in production
if (process.env.NODE_ENV === 'production') {
  logger.add(new CloudWatchTransport({
    logGroupName: `/ecs/ploy-${process.env.NODE_ENV}`,
    logStreamName: `api-${new Date().toISOString().slice(0, 10)}`,
    awsRegion: process.env.AWS_REGION || 'us-east-1'
  }));
}

export default logger;
```

---

## Security Configuration

### Secrets Management

```hcl
# terraform/secrets.tf
# Secrets Manager
resource "aws_secretsmanager_secret" "jwt" {
  name                    = "ploy/${var.environment}/jwt-secret"
  description             = "JWT secret for Ploy application"
  recovery_window_in_days = 7

  tags = {
    Name = "ploy-${var.environment}-jwt-secret"
  }
}

resource "aws_secretsmanager_secret_version" "jwt" {
  secret_id     = aws_secretsmanager_secret.jwt.id
  secret_string = var.jwt_secret
}

resource "aws_secretsmanager_secret" "redis_auth" {
  name                    = "ploy/${var.environment}/redis-auth-token"
  description             = "Redis auth token for Ploy application"
  recovery_window_in_days = 7

  tags = {
    Name = "ploy-${var.environment}-redis-auth"
  }
}

resource "aws_secretsmanager_secret_version" "redis_auth" {
  secret_id     = aws_secretsmanager_secret.redis_auth.id
  secret_string = var.redis_auth_token
}

resource "aws_secretsmanager_secret" "blockchain_keys" {
  name                    = "ploy/${var.environment}/blockchain-keys"
  description             = "Blockchain private keys for Ploy application"
  recovery_window_in_days = 7

  tags = {
    Name = "ploy-${var.environment}-blockchain-keys"
  }
}

resource "aws_secretsmanager_secret_version" "blockchain_keys" {
  secret_id = aws_secretsmanager_secret.blockchain_keys.id
  secret_string = jsonencode({
    sui_private_key     = var.sui_private_key
    solana_private_key  = var.solana_private_key
    ethereum_private_key = var.ethereum_private_key
  })
}

# IAM policy for ECS to access secrets
resource "aws_iam_policy" "ecs_secrets" {
  name        = "ploy-${var.environment}-ecs-secrets"
  description = "Policy for ECS to access Secrets Manager"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          aws_secretsmanager_secret.jwt.arn,
          aws_secretsmanager_secret.redis_auth.arn,
          aws_secretsmanager_secret.blockchain_keys.arn
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_secrets" {
  role       = aws_iam_role.ecs_task.name
  policy_arn = aws_iam_policy.ecs_secrets.arn
}

# Variables for secrets
variable "jwt_secret" {
  description = "JWT secret"
  type        = string
  sensitive   = true
}

variable "sui_private_key" {
  description = "SUI private key"
  type        = string
  sensitive   = true
}

variable "solana_private_key" {
  description = "Solana private key"
  type        = string
  sensitive   = true
}

variable "ethereum_private_key" {
  description = "Ethereum private key"
  type        = string
  sensitive   = true
}
```

### SSL Certificate Management

```hcl
# terraform/ssl.tf
# ACM Certificate
resource "aws_acm_certificate" "main" {
  domain_name               = var.domain_name
  subject_alternative_names = [
    "*.${var.domain_name}",
    "www.${var.domain_name}"
  ]
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "ploy-${var.environment}-cert"
  }
}

# Certificate validation
resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# Route53 records for certificate validation
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.main.zone_id
}

data "aws_route53_zone" "main" {
  name         = var.domain_name
  private_zone = false
}
```

---

## Scaling & Performance

### Auto Scaling Configuration

```hcl
# terraform/autoscaling.tf
# Application Auto Scaling Target for API
resource "aws_appautoscaling_target" "api" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# Auto Scaling Policy for API - CPU
resource "aws_appautoscaling_policy" "api_cpu" {
  name               = "ploy-${var.environment}-api-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  service_namespace  = aws_appautoscaling_target.api.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

# Auto Scaling Policy for API - Memory
resource "aws_appautoscaling_policy" "api_memory" {
  name               = "ploy-${var.environment}-api-memory-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  service_namespace  = aws_appautoscaling_target.api.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }
    target_value = 80.0
  }
}

# Application Auto Scaling Target for Web
resource "aws_appautoscaling_target" "web" {
  max_capacity       = 6
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.web.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# Auto Scaling Policy for Web - CPU
resource "aws_appautoscaling_policy" "web_cpu" {
  name               = "ploy-${var.environment}-web-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.web.resource_id
  scalable_dimension = aws_appautoscaling_target.web.scalable_dimension
  service_namespace  = aws_appautoscaling_target.web.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

### Performance Optimization

```typescript
// src/middleware/performance.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import compression from 'compression';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

@Injectable()
export class PerformanceMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Enable gzip compression
    compression({
      level: 6,
      threshold: 1024,
      filter: (req, res) => {
        if (req.headers['x-no-compression']) {
          return false;
        }
        return compression.filter(req, res);
      }
    })(req, res, () => {
      // Security headers
      helmet({
        contentSecurityPolicy: {
          directives: {
            defaultSrc: ["'self'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
            scriptSrc: ["'self'"],
            imgSrc: ["'self'", "data:", "https:"],
            connectSrc: ["'self'", "https://api.ploy.app"],
            fontSrc: ["'self'", "https://fonts.gstatic.com"],
            objectSrc: ["'none'"],
            mediaSrc: ["'self'"],
            frameSrc: ["'none'"],
          },
        },
      })(req, res, () => {
        // Rate limiting
        const limiter = rateLimit({
          windowMs: 15 * 60 * 1000, // 15 minutes
          max: 1000, // limit each IP to 1000 requests per windowMs
          message: 'Too many requests from this IP',
          standardHeaders: true,
          legacyHeaders: false,
        });
        
        limiter(req, res, next);
      });
    });
  }
}
```

---

## Deployment Commands

### Terraform Variables

```hcl
# terraform/production.tfvars
aws_region = "us-east-1"
environment = "production"
domain_name = "ploy.app"

# Cloudflare
cloudflare_zone_id = "your-cloudflare-zone-id"

# Database
db_password = "your-secure-db-password"
redis_auth_token = "your-redis-auth-token"

# Secrets
jwt_secret = "your-jwt-secret"
sui_private_key = "your-sui-private-key"
solana_private_key = "your-solana-private-key"
ethereum_private_key = "your-ethereum-private-key"

# Monitoring
alert_email = "alerts@ploy.app"
```

### Deployment Steps

```bash
# 1. Initialize and deploy infrastructure
cd terraform
terraform init
terraform plan -var-file="production.tfvars"
terraform apply -var-file="production.tfvars"

# 2. Build and push Docker images
./scripts/build-and-push.sh production

# 3. Deploy application
./scripts/deploy.sh production

# 4. Verify deployment
curl -f https://api.ploy.app/health
curl -f https://ploy.app
```

This comprehensive deployment guide provides a production-ready setup using AWS ECS Fargate, CloudFront, and Cloudflare with all necessary security, monitoring, and scaling configurations.
