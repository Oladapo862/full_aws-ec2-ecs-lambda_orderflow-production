# AWS Production CI/CD & Event-Driven Order Management Platform

A hands-on AWS production engineering project demonstrating how to build, deploy, automate, monitor, secure, and troubleshoot a containerized application across **EC2, ECS/Fargate, and Lambda**, with an event-driven architecture using **SQS, DynamoDB, S3, EventBridge, and SNS**.

The project is designed to demonstrate practical **AWS infrastructure, DevOps, CI/CD, IAM, networking, containerization, serverless, observability, and production troubleshooting skills** rather than simply deploying a basic application.

---

# Project Overview

This project is an **Order Management System** built with Python and Flask.

The same core application/business logic is deployed using multiple AWS compute models:

```text
                    GitHub Repository
                           │
                           ▼
                    GitHub Actions
                           │
                          OIDC
                           │
                           ▼
                          AWS
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
            EC2           ECS          Lambda
         Production    Production     Serverless
             │             │             │
             └─────────────┼─────────────┘
                           │
                         RDS
                           │
                    Secrets Manager
```

The project then extends Lambda into an event-driven architecture:

```text
                         API Gateway
                              │
                              ▼
                       Submit Lambda
                              │
                              ▼
                             SQS
                              │
                              ▼
                       Process Lambda
                         ┌────┼────┐
                         │    │    │
                         ▼    ▼    ▼
                    DynamoDB  S3  EventBridge
                                      │
                                      ▼
                                     SNS
                                      │
                                      ▼
                                    Email
```

The deployment process is automated through GitHub Actions:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ▼
GitHub OIDC
    │
    ▼
AWS IAM Role
    │
    ├──────────────► ECR
    │
    ├──────────────► ECS
    │
    ├──────────────► Lambda
    │
    └──────────────► EC2 through SSM
```

---

# What I Built

This project covers the complete lifecycle of a production-style AWS workload:

* Built a Python Flask order management application
* Created reusable application/business logic
* Containerized the application with Docker
* Ran the application on EC2
* Ran the application on ECS Fargate
* Deployed serverless versions using Lambda container images
* Created an Application Load Balancer
* Configured target groups and health checks
* Created a production VPC
* Created public and private subnets across multiple Availability Zones
* Configured Internet Gateway and routing
* Configured private network connectivity
* Created security groups for individual application layers
* Deployed MySQL using Amazon RDS
* Stored database credentials in AWS Secrets Manager
* Used IAM roles instead of hardcoded AWS credentials
* Used AWS Systems Manager Session Manager to access EC2
* Created Amazon ECR repositories for container images
* Built an event-driven architecture with SQS
* Connected SQS to Lambda
* Stored processed orders in DynamoDB
* Stored objects/data in S3
* Published events through EventBridge
* Sent notifications using SNS
* Configured CloudWatch logging and monitoring
* Built production troubleshooting scripts
* Used CloudWatch Logs Insights for log investigation
* Implemented GitHub Actions CI/CD
* Configured GitHub OIDC authentication
* Automated ECS deployments
* Automated Lambda deployments
* Designed an EC2 deployment mechanism using SSM
* Practiced least-privilege IAM
* Built a structured production incident troubleshooting methodology

---

# Application

The application is a simple Order Management System designed specifically to provide a realistic workload for practicing AWS production engineering.

The application exposes endpoints such as:

```text
GET  /
GET  /health
GET  /db
POST /submit
```

## Health Endpoint

```text
GET /health
```

Returns:

```json
{
  "status": "healthy"
}
```

This endpoint is used by:

* Local troubleshooting
* Docker testing
* ALB health checks
* ECS target health checks
* Production validation

---

# Database Health

The application also provides database connectivity testing.

```text
GET /db
```

The application connects to MySQL through:

```text
Application
    │
    ▼
Secrets Manager
    │
    ▼
Database credentials
    │
    ▼
Amazon RDS MySQL
```

Database credentials are not hardcoded into the application.

---

# Application Structure

The application separates business logic from the deployment environment.

```text
app/
├── __init__.py
├── app.py
├── db.py
├── orders.py
├── lambda_submit.py
└── lambda_process.py
```

## `app.py`

Contains the Flask HTTP application.

## `db.py`

Handles:

* Secrets Manager access
* Database credentials
* RDS connection

## `orders.py`

Contains reusable order business logic.

This allows the same business logic to be used across:

```text
EC2
ECS
Lambda
```

## `lambda_submit.py`

Provides the Lambda entry point for submitting orders.

## `lambda_process.py`

Provides the Lambda entry point for processing orders.

---

# Docker

The application is containerized using Docker.

The Docker image contains:

```text
Python
Flask
Gunicorn
PyMySQL
boto3
Application code
```

The application runs on:

```text
0.0.0.0:8000
```

Gunicorn is used as the production application server.

The container architecture is:

```text
Docker
   │
   ▼
Gunicorn
   │
   ▼
Flask
   │
   ▼
Application Logic
```

The same containerized application can be deployed to:

```text
EC2
ECS/Fargate
```

---

# AWS Network Architecture

A dedicated VPC was designed for the production environment.

```text
VPC
10.0.0.0/16
│
├── Availability Zone A
│   ├── Public Subnet
│   └── Private Subnet
│
└── Availability Zone B
    ├── Public Subnet
    └── Private Subnet
```

The design separates internet-facing infrastructure from private application and database resources.

---

# Public Subnets

Public subnets are used for resources that need to receive internet traffic, such as the Application Load Balancer.

```text
Internet
    │
    ▼
Internet Gateway
    │
    ▼
Public Subnet
    │
    ▼
Application Load Balancer
```

---

# Private Subnets

Private subnets are used for application and database resources.

Examples include:

```text
EC2
ECS Tasks
RDS
```

The architecture avoids exposing application servers and databases directly to the public internet.

---

# Internet Gateway

An Internet Gateway provides internet connectivity for the public portion of the architecture.

```text
Internet
    │
    ▼
Internet Gateway
    │
    ▼
Public Route Table
    │
    ▼
Public Subnets
```

---

# NAT / Private Connectivity

Private workloads can use appropriate outbound connectivity when they need access to the general internet.

AWS service communication can use VPC endpoints where appropriate.

The architecture therefore distinguishes between:

```text
AWS services
     │
     ▼
VPC Endpoints
```

and:

```text
General internet traffic
     │
     ▼
NAT Gateway
```

---

# VPC Endpoints

Private workloads can access AWS services through VPC endpoints.

Services considered in the architecture include:

```text
SSM
SSM Messages
EC2 Messages
Secrets Manager
S3
```

This reduces the need for private application resources to rely on public internet paths for AWS service communication.

---

# Security Groups

Separate security groups are used to control communication between application layers.

```text
sg-alb
sg-ec2
sg-ecs
sg-rds
sg-vpc-endpoint
sg-efs
```

The intended traffic flow is:

```text
Internet
   │
   ▼
ALB :80/:443
   │
   ▼
EC2/ECS :8000
   │
   ▼
RDS :3306
```

The database is not opened to the internet.

---

# Application Load Balancer

An Application Load Balancer is used as the public entry point for the containerized application.

```text
Internet
    │
    ▼
Application Load Balancer
    │
    ▼
Target Group
    │
    ▼
EC2 / ECS
    │
    ▼
Application :8000
```

The ALB performs health checks against:

```text
/health
```

on port:

```text
8000
```

---

# ALB Health Checks

The health check verifies that the application is actually responding.

```text
ALB
 │
 ├── HTTP
 ├── Port 8000
 └── /health
       │
       ▼
Application
```

A healthy response is:

```text
HTTP 200
```

with:

```json
{
  "status": "healthy"
}
```

This provides a clear distinction between:

```text
Network reachable
```

and:

```text
Application actually healthy
```

---

# Amazon RDS

The project uses Amazon RDS MySQL as the relational database.

The database is placed in private subnets.

```text
EC2
 │
 ├──────────────┐
 │              │
 ▼              ▼
Secrets       RDS
Manager       MySQL
 │
 └── credentials
```

The database stores order information such as:

```text
id
customer_name
product
status
created_at
```

---

# Database Security

The RDS security group allows MySQL traffic only from approved application workloads.

```text
EC2 ───────► RDS :3306
ECS ───────► RDS :3306
```

The database does not accept:

```text
Internet → RDS :3306
```

---

# AWS Secrets Manager

Database credentials are stored in AWS Secrets Manager.

Secret:

```text
production/order-db
```

Example structure:

```json
{
  "host": "RDS-ENDPOINT",
  "username": "admin",
  "password": "DATABASE-PASSWORD",
  "dbname": "orders",
  "port": 3306
}
```

The application retrieves the secret through the AWS SDK.

This avoids putting credentials directly into:

* Source code
* Dockerfiles
* GitHub repositories
* ECS task definitions
* Lambda source code

---

# IAM

IAM is used to control what each AWS component can access.

The project uses separate roles for different workloads.

Examples:

```text
order-ec2-production-role
order-ecs-execution-role
order-ecs-task-role
order-submit-role
order-process-role
github-actions-production-role
```

The objective is to follow the principle of:

```text
Least Privilege
```

Each role should have only the permissions required for its responsibility.

---

# EC2 Production

The application is deployed to an EC2 instance.

The architecture is:

```text
ALB
 │
 ▼
EC2
 │
 ▼
Docker
 │
 ▼
Gunicorn
 │
 ▼
Flask
 │
 ▼
RDS
```

The EC2 instance runs the containerized application on port:

```text
8000
```

---

# EC2 Access with Systems Manager

Instead of depending on direct SSH access, the project uses AWS Systems Manager Session Manager.

```text
AWS Console
     │
     ▼
Session Manager
     │
     ▼
EC2
```

This provides administrative access without requiring the application server to expose SSH publicly.

---

# EC2 Docker Deployment

The application can be built and started on EC2 using Docker.

Example deployment flow:

```bash
docker build -t order-management .
```

Then:

```bash
docker run -d \
  --name order-app \
  -p 8000:8000 \
  order-management
```

Application validation:

```bash
curl http://127.0.0.1:8000/health
```

Expected:

```json
{
  "status": "healthy"
}
```

---

# Amazon ECR

Amazon Elastic Container Registry is used to store Docker images.

Example repository:

```text
order-management
```

The deployment flow is:

```text
Developer
   │
   ▼
Docker build
   │
   ▼
Docker image
   │
   ▼
ECR
```

Images can then be consumed by ECS and Lambda container deployments.

---

# ECS / Fargate

The same application is also deployed to Amazon ECS using Fargate.

Architecture:

```text
ALB
 │
 ▼
ECS Target Group
 │
 ▼
Fargate Task
 │
 ▼
Container :8000
 │
 ▼
Gunicorn
 │
 ▼
Flask
```

The ECS service runs the container without requiring management of EC2 instances for the ECS workload.

---

# ECS Cluster

Example cluster:

```text
order-production-cluster
```

The cluster hosts the application service.

Example service:

```text
order-production-service
```

---

# ECS Task Definition

The ECS task definition defines:

* Container image
* CPU
* Memory
* Container port
* Execution role
* Task role
* CloudWatch logging
* Secrets Manager references

The container listens on:

```text
8000
```

---

# ECS Execution Role vs Task Role

A key part of the implementation is understanding the difference between the two roles.

## Execution Role

The ECS execution role allows ECS to perform infrastructure-level operations such as:

```text
Pull image from ECR
Send container logs to CloudWatch
```

## Task Role

The ECS task role represents the application itself.

It is used when the application needs AWS API access, such as:

```text
Secrets Manager
DynamoDB
S3
EventBridge
```

The distinction is:

```text
Execution Role
      │
      ▼
ECS infrastructure

Task Role
      │
      ▼
Application running inside container
```

---

# Lambda

The project also implements the application using AWS Lambda.

Lambda uses container images to package the application and its dependencies.

Two Lambda functions are used conceptually:

```text
order-submit
order-process
```

The architecture becomes:

```text
API Gateway
     │
     ├──────────────► order-submit Lambda
     │
     └──────────────► order-process Lambda
```

---

# API Gateway

API Gateway provides HTTP access to Lambda.

Example routes:

```text
POST /submit
POST /process
```

Routing:

```text
POST /submit
      │
      ▼
order-submit Lambda
```

and:

```text
POST /process
      │
      ▼
order-process Lambda
```

---

# Event-Driven Architecture

The project extends the Lambda implementation into an asynchronous event-driven system.

The architecture is:

```text
API Gateway
     │
     ▼
Submit Lambda
     │
     ▼
SQS
     │
     ▼
Process Lambda
     │
     ├──────────────► DynamoDB
     │
     ├──────────────► S3
     │
     └──────────────► EventBridge
                           │
                           ▼
                          SNS
                           │
                           ▼
                         Email
```

This separates the initial request from background processing.

---

# Amazon SQS

The queue:

```text
order-processing
```

is used to decouple order submission from order processing.

Instead of:

```text
Client
 ↓
Lambda
 ↓
Database
 ↓
Response
```

the asynchronous architecture becomes:

```text
Client
 ↓
Submit Lambda
 ↓
SQS
 ↓
Response

SQS
 ↓
Process Lambda
 ↓
Processing
```

This improves decoupling and allows the processing workload to scale independently.

---

# SQS → Lambda

The Process Lambda is connected to SQS using an event source mapping.

A Lambda invocation receives records in an SQS event structure similar to:

```json
{
  "Records": [
    {
      "body": "{\"order_id\":123}"
    }
  ]
}
```

The Lambda processes the messages contained in:

```text
event["Records"]
```

and reads each:

```text
record["body"]
```

---

# DynamoDB

Processed order information can be stored in DynamoDB.

Example table:

```text
orders
```

Partition key:

```text
order_id
```

Architecture:

```text
SQS
 ↓
Process Lambda
 ↓
DynamoDB
```

This demonstrates using a NoSQL database for serverless workloads alongside RDS.

---

# Amazon S3

S3 is used for object/data storage.

Architecture:

```text
Process Lambda
      │
      ▼
     S3
```

The Lambda execution role is granted only the required S3 permissions.

---

# EventBridge

After processing an order, the application can publish an event.

Example event:

```json
{
  "Source": "order.management",
  "DetailType": "OrderProcessed",
  "Detail": {
    "order_id": 123,
    "status": "PROCESSED"
  }
}
```

Architecture:

```text
Process Lambda
      │
      ▼
EventBridge
```

This separates event production from event consumers.

---

# SNS

Amazon SNS is used for notifications.

The architecture is:

```text
Process Lambda
      │
      ▼
EventBridge
      │
      ▼
SNS
      │
      ▼
Email notification
```

This demonstrates an event-driven notification pipeline.

---

# CloudWatch Monitoring

CloudWatch is used as the main observability platform.

The project monitors:

```text
EC2
ECS
Lambda
ALB
Application
```

Important signals include:

```text
CPU
Memory
Disk
Request errors
Latency
Lambda errors
Lambda duration
SQS backlog
Target health
Application logs
Container logs
```

---

# CloudWatch Logs

Application logs are centralized into CloudWatch.

For ECS:

```text
Container
    │
    ▼
awslogs
    │
    ▼
CloudWatch
```

Lambda automatically produces invocation logs.

Example Lambda log groups:

```text
/aws/lambda/order-submit
/aws/lambda/order-process
```

ECS:

```text
/ecs/order-production
```

---

# CloudWatch Logs Insights

CloudWatch Logs Insights is used to investigate production incidents.

Basic query:

```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 50
```

Error search:

```sql
fields @timestamp, @message
| filter @message like /ERROR|Exception|Traceback/
| sort @timestamp desc
| limit 100
```

Broader failure search:

```sql
fields @timestamp, @message
| filter @message like /ERROR|Exception|Traceback|failed|Failure|timeout|timed out|AccessDenied|Throttl|OOM|Killed|refused/
| sort @timestamp desc
| limit 200
```

The goal is to move from:

```text
Something is broken
```

to:

```text
This service failed
      ↓
At this time
      ↓
With this error
      ↓
Because of this dependency
```

---

# Production Troubleshooting

A major part of this project is not just deploying the infrastructure, but learning how to **troubleshoot it when it fails**.

The troubleshooting methodology follows:

```text
INCIDENT
   │
   ▼
CAN THE APPLICATION BE REACHED?
   │
   ├── NO
   │    │
   │    ├── Is the application running?
   │    ├── Is Docker running?
   │    ├── Is Gunicorn running?
   │    ├── Is the port listening?
   │    ├── Is the network reachable?
   │    └── Is the ALB healthy?
   │
   └── YES
        │
        ▼
    WHAT RESPONSE?
        │
        ├── 4XX
        ├── 5XX
        ├── TIMEOUT
        └── SLOW
             │
             ▼
          READ LOGS
             │
             ▼
       IDENTIFY FAILURE
             │
             ▼
      FOLLOW DEPENDENCY
             │
             ▼
       PROVE ROOT CAUSE
```

---

# Troubleshooting Scripts

The project includes a set of production troubleshooting scripts.

```text
prod-checks/
├── 01-resources.sh
├── 02-app-health.sh
├── 03-services-logs.sh
├── 04-network-rds.sh
├── 05-alb-target.sh
├── 06-aws-access.sh
└── 07-performance.sh
```

---

# 1. Resource Troubleshooting

The resource script checks:

```text
CPU
Memory
Disk
Load
```

Commands include:

```bash
nproc
uptime
free -h
df -h
```

This answers:

> Is the server under resource pressure?

---

# 2. Application Health Troubleshooting

The application health script checks:

```text
/health
/db
ALB /health
```

Using:

```bash
curl
```

This helps distinguish:

```text
Application failure
```

from:

```text
ALB/network failure
```

---

# 3. Services and Logs

The services script investigates:

```text
Docker
Containers
Application logs
Gunicorn
```

Important commands include:

```bash
systemctl status docker
docker ps -a
docker logs --tail 50 CONTAINER
journalctl
```

For a Dockerized deployment, container logs are the primary source for application/Gunicorn output when Gunicorn runs inside the container.

---

# 4. Network and RDS Troubleshooting

The network script checks:

```text
IP configuration
Routing
Listening ports
DNS
RDS port connectivity
```

Important commands include:

```bash
ip addr
ip route
ss -lntp
getent hosts HOST
nc -vz HOST 3306
```

This creates a clear troubleshooting sequence:

```text
DNS
 ↓
Routing
 ↓
TCP connectivity
 ↓
Port
 ↓
Application
```

---

# 5. ALB and Target Health

AWS CLI is used to inspect target health.

Example:

```bash
aws elbv2 describe-target-health \
    --target-group-arn TARGET_GROUP_ARN
```

This helps investigate states such as:

```text
healthy
unhealthy
initial
draining
unused
```

The target health reason is particularly useful for determining why the ALB cannot successfully reach the application.

---

# 6. AWS Access Troubleshooting

The project also checks AWS identity and access.

Example:

```bash
aws sts get-caller-identity
```

This answers:

> Which IAM identity is making this AWS API request?

The troubleshooting process for an authorization failure is:

```text
AccessDenied
     │
     ▼
Identify IAM principal
     │
     ▼
Identify AWS action
     │
     ▼
Identify resource
     │
     ▼
Check IAM policy
     │
     ▼
Check resource policy / conditions
     │
     ▼
Verify permission
```

---

# 7. Performance Troubleshooting

Performance checks include:

```bash
uptime
ps aux --sort=-%cpu
ps aux --sort=-%mem
ss -s
ss -antp
time curl ...
```

This helps investigate:

```text
High CPU
High memory
High connection count
Slow requests
Worker problems
```

---

# Production Incident Troubleshooting Model

The troubleshooting process is based on a layered approach.

```text
                 PRODUCTION ALERT
                         │
                         ▼
                CAN APP BE REACHED?
                   ┌─────┴─────┐
                   │           │
                  YES          NO
                   │           │
                   ▼           ▼
             WHAT RESPONSE?  IS APP UP?
                   │           │
          ┌────────┼───────┐   ├── YES
          │        │       │   │
         4XX      5XX   TIMEOUT│
          │        │       │   └── NO
          ▼        ▼       ▼        │
       APP/API   LOGS   NETWORK     ▼
       /AUTH     /DB    /ALB      LOGS
          │        │       │
          └────────┼───────┘
                   ▼
             FIND ROOT CAUSE
```

The principle is:

```text
Don't guess.
Check.
Measure.
Read logs.
Follow dependencies.
Prove the failure.
```

---

# Common Production Failure Areas Practiced

The infrastructure and troubleshooting work covers failures involving:

## Application

```text
Application not responding
Application crashes
HTTP 500
HTTP 404
Slow responses
Health endpoint failure
Database endpoint failure
```

## Docker

```text
Container stopped
Container repeatedly restarting
Incorrect port mapping
Application not listening
Container startup failure
Container logs
```

## Gunicorn

```text
Worker failure
Worker timeout
Application startup failure
Incorrect binding
Application import errors
```

## ALB

```text
Unhealthy targets
Health check failure
Incorrect health-check path
Incorrect port
Target draining
Target connectivity
```

## Network

```text
DNS failure
Route failure
Port unreachable
Security group restrictions
Private subnet connectivity
VPC endpoint connectivity
```

## RDS

```text
Database unreachable
Port 3306 failure
Authentication failure
Incorrect credentials
Incorrect database name
Connection timeout
```

## IAM

```text
AccessDenied
Missing IAM permission
Incorrect execution role
Incorrect task role
Incorrect resource ARN
```

## Secrets Manager

```text
Secret not found
Invalid secret format
Incorrect secret name
AccessDenied
Invalid credentials
Missing database fields
```

## ECS

```text
Task fails to start
Task stops
Container health failure
ECR image pull failure
Execution role failure
Task role failure
Secret retrieval failure
```

## Lambda

```text
Function errors
Timeout
Permission errors
Dependency errors
Cold-start related latency
Memory pressure
SQS processing failure
```

## SQS

```text
Message backlog
Old messages
Lambda processing failure
Consumer errors
Processing delays
```

---

# CI/CD

The project implements CI/CD using GitHub Actions.

The intended pipeline is:

```text
Developer
    │
    ▼
git push
    │
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Checkout
    │
    ├── Authenticate with AWS using OIDC
    │
    ├── Build Docker image
    │
    ├── Push image to ECR
    │
    ├── Update ECS
    │
    ├── Update Lambda
    │
    └── Deploy to EC2 through SSM
```

---

# GitHub OIDC

The project uses GitHub's OIDC integration instead of storing long-lived AWS access keys inside GitHub.

Architecture:

```text
GitHub Actions
      │
      ▼
GitHub OIDC
      │
      ▼
AWS STS
      │
      ▼
IAM Role
      │
      ▼
Temporary AWS credentials
```

This improves the security of the CI/CD pipeline.

---

# GitHub Actions Permissions

The deployment role is designed around the principle of least privilege.

It requires only the AWS permissions necessary to deploy the workloads.

Examples include permissions related to:

```text
ECR
ECS
Lambda
SSM
PassRole
```

The project intentionally avoids using:

```text
AdministratorAccess
```

as a shortcut for CI/CD.

---

# Container Image Versioning

The CI/CD pipeline can tag images using the Git commit SHA.

Example:

```text
order-management:COMMIT_SHA
```

This provides traceability between:

```text
Git commit
     │
     ▼
Docker image
     │
     ▼
Deployment
```

Instead of relying only on:

```text
latest
```

---

# ECS Deployment

The ECS pipeline performs:

```text
GitHub
  ↓
Build image
  ↓
Push to ECR
  ↓
Render new ECS task definition
  ↓
Deploy new task definition
  ↓
ECS service replaces old task
  ↓
Wait for service stability
```

This provides an automated container deployment process.

---

# Lambda Deployment

Lambda container images are also stored in ECR.

The deployment flow is:

```text
GitHub Actions
      │
      ▼
Build Lambda image
      │
      ▼
Push image to ECR
      │
      ▼
Update Lambda function
```

Functions include:

```text
order-submit
order-process
```

---

# EC2 Deployment

EC2 uses Systems Manager rather than requiring GitHub Actions to SSH directly into the instance.

Conceptually:

```text
GitHub Actions
      │
      ▼
AWS SSM
      │
      ▼
EC2
      │
      ▼
Pull/build/restart application
```

This keeps direct SSH access out of the CI/CD architecture.

---

# Repository Structure

The repository is organized around application code, infrastructure, and automation.

```text
order-management-cicd/
│
├── app/
│   ├── __init__.py
│   ├── app.py
│   ├── db.py
│   ├── orders.py
│   ├── lambda_submit.py
│   └── lambda_process.py
│
├── frontend/
│   └── index.html
│
├── infrastructure/
│   ├── iam/
│   ├── ecs/
│   ├── lambda/
│   └── event-driven/
│
├── prod-checks/
│   ├── 01-resources.sh
│   ├── 02-app-health.sh
│   ├── 03-services-logs.sh
│   ├── 04-network-rds.sh
│   ├── 05-alb-target.sh
│   ├── 06-aws-access.sh
│   └── 07-performance.sh
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── Dockerfile
├── Dockerfile.lambda
├── requirements.txt
├── .dockerignore
├── .gitignore
└── README.md
```

---

# Technology Stack

## Application

```text
Python
Flask
Gunicorn
PyMySQL
boto3
```

## Containers

```text
Docker
Amazon ECR
```

## Compute

```text
Amazon EC2
Amazon ECS
AWS Fargate
AWS Lambda
```

## Networking

```text
Amazon VPC
Public Subnets
Private Subnets
Internet Gateway
NAT Gateway
VPC Endpoints
Security Groups
Application Load Balancer
```

## Database and Storage

```text
Amazon RDS MySQL
Amazon DynamoDB
Amazon S3
AWS Secrets Manager
```

## Event-Driven Services

```text
Amazon SQS
Amazon EventBridge
Amazon SNS
```

## Monitoring

```text
Amazon CloudWatch
CloudWatch Logs
CloudWatch Logs Insights
```

## Security

```text
AWS IAM
IAM Roles
GitHub OIDC
AWS Systems Manager
```

## CI/CD

```text
GitHub
GitHub Actions
GitHub OIDC
Amazon ECR
Amazon ECS
AWS Lambda
AWS Systems Manager
```

---

# Key Skills Demonstrated

This project demonstrates practical experience with:

### AWS Infrastructure

* VPC design
* Public/private subnet architecture
* Availability Zones
* Route tables
* Internet Gateway
* NAT Gateway
* VPC endpoints
* Security groups
* Application Load Balancer

### Compute

* EC2
* ECS
* Fargate
* Lambda
* Docker
* Lambda container images

### Databases

* RDS MySQL
* DynamoDB
* Database connectivity troubleshooting
* Database security

### Security

* IAM roles
* Least privilege
* Secrets Manager
* GitHub OIDC
* Systems Manager Session Manager
* Security groups

### DevOps

* Docker image creation
* ECR
* Git
* GitHub Actions
* Automated deployments
* Immutable image tagging
* ECS deployment automation
* Lambda deployment automation
* EC2 deployment through SSM

### Serverless

* API Gateway
* Lambda
* SQS
* DynamoDB
* S3
* EventBridge
* SNS

### Observability

* CloudWatch
* CloudWatch Logs
* CloudWatch Logs Insights
* Application logging
* Container logging
* Lambda logging
* Metrics-based troubleshooting

### Production Troubleshooting

* CPU investigation
* Memory investigation
* Disk investigation
* Application health checks
* Docker troubleshooting
* Gunicorn troubleshooting
* Network troubleshooting
* DNS troubleshooting
* Port troubleshooting
* ALB target health
* RDS connectivity
* IAM authorization failures
* Secrets Manager failures
* ECS task failures
* Lambda failures
* SQS backlog investigation

---

# Production Troubleshooting Philosophy

The most important skill demonstrated by this project is not memorizing AWS commands.

It is learning how to systematically isolate failures.

The troubleshooting model is:

```text
                    INCIDENT
                       │
                       ▼
                CAN USER REACH IT?
                       │
             ┌─────────┴─────────┐
             │                   │
            YES                  NO
             │                   │
             ▼                   ▼
        HTTP RESPONSE       IS SERVICE UP?
             │                   │
       ┌─────┼─────┐             │
       │     │     │             ▼
      4XX   5XX  TIMEOUT       SERVICE
       │     │     │            CHECK
       └─────┼─────┘             │
             │                   ▼
             ▼                 LOGS
           LOGS                  │
             │                   ▼
             ▼               ROOT CAUSE
       FIND ERROR
             │
             ▼
       FIND DEPENDENCY
             │
             ▼
        CHECK METRICS
             │
             ▼
       PROVE ROOT CAUSE
```

The goal is to move from:

```text
"Users are complaining."
```

to:

```text
"The application returned HTTP 500 because the ECS task could not retrieve the database secret due to a missing Secrets Manager permission on the task role."
```

That is the difference between simply operating AWS resources and actually troubleshooting production systems.

---

# Example Production Investigation

Suppose users report:

```text
The application is returning HTTP 500.
```

The investigation becomes:

```text
1. curl /health
        ↓
   Application reachable

2. Check HTTP response
        ↓
   HTTP 500

3. Open CloudWatch Logs Insights
        ↓
   Search ERROR / Exception

4. Find:
   AccessDeniedException
        ↓
5. Identify dependency:
   Secrets Manager
        ↓
6. Identify principal:
   ECS task role
        ↓
7. Check:
   secretsmanager:GetSecretValue
        ↓
8. Verify secret ARN
        ↓
9. Correct IAM permission
        ↓
10. Test application again
```

This approach avoids random changes and focuses on evidence.

---

# What This Project Demonstrates

This project goes beyond:

```text
"Here is my Flask application running on AWS."
```

It demonstrates the full production lifecycle:

```text
BUILD
  ↓
CONTAINERIZE
  ↓
NETWORK
  ↓
SECURE
  ↓
DEPLOY
  ↓
MONITOR
  ↓
TROUBLESHOOT
  ↓
AUTOMATE
  ↓
SCALE
  ↓
EVENT-DRIVE
```

It also demonstrates the ability to work with multiple AWS compute models and understand when their operational models differ:

```text
EC2
 ↓
Manage the server

ECS/Fargate
 ↓
Manage the container workload

Lambda
 ↓
Manage the function/event workload
```

---

# Project Outcome

The completed platform provides three deployment models for the same application:

```text
                  ORDER MANAGEMENT APPLICATION
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
             EC2             ECS            Lambda
          Dockerized       Fargate        Serverless
              │               │               │
              └───────────────┼───────────────┘
                              │
                           AWS DATA
                              │
                 ┌────────────┼────────────┐
                 │            │            │
                 ▼            ▼            ▼
                RDS        DynamoDB        S3
                              │
                              ▼
                         EVENTBRIDGE
                              │
                              ▼
                             SNS
                              │
                              ▼
                            EMAIL
```

And the deployment lifecycle is automated:

```text
Developer
    │
    ▼
Git push
    │
    ▼
GitHub Actions
    │
    ▼
OIDC
    │
    ▼
AWS
    │
    ├──► ECR
    ├──► ECS
    ├──► Lambda
    └──► EC2/SSM
```

---

# Final Summary

This repository represents a hands-on AWS production engineering project focused on building and operating a real application across multiple AWS architectures.

The project combines:

```text
AWS Infrastructure
        +
Docker
        +
EC2
        +
ECS/Fargate
        +
Lambda
        +
RDS
        +
DynamoDB
        +
S3
        +
SQS
        +
EventBridge
        +
SNS
        +
IAM
        +
Secrets Manager
        +
CloudWatch
        +
GitHub Actions
        +
GitHub OIDC
        +
Systems Manager
        +
Production Troubleshooting
```

The objective is not simply to demonstrate that an application can run.

The objective is to demonstrate the ability to:

```text
BUILD IT
   ↓
DEPLOY IT
   ↓
SECURE IT
   ↓
MONITOR IT
   ↓
AUTOMATE IT
   ↓
TROUBLESHOOT IT
   ↓
IMPROVE IT
```

This project therefore serves as a practical portfolio demonstrating **AWS Cloud Engineering, DevOps, CI/CD, containerization, serverless architecture, event-driven systems, IAM/security, observability, and production incident troubleshooting**.
