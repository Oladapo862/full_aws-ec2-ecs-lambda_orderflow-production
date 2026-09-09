# Production Flask Application — AWS EC2, ECS/Fargate & EKS

A production-style Flask application deployed across multiple AWS compute platforms using **Terraform**, with shared AWS infrastructure for networking, database, secrets, storage, monitoring, logging, container images, and alerting.

The same Flask application can run on:

* **Amazon EC2 + Auto Scaling Group**
* **Amazon ECS with Fargate**
* **Amazon EKS with Kubernetes**

All deployment models use the same common AWS infrastructure.

---

## Architecture

```text
                              INTERNET
                                  │
                                  ▼
                       ┌────────────────────┐
                       │        ALB         │
                       │  Application Load  │
                       │      Balancer      │
                       └─────────┬──────────┘
                                 │
                ┌────────────────┼────────────────┐
                │                │                │
                ▼                ▼                ▼
        ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
        │     EC2     │  │ ECS/Fargate  │  │     EKS      │
        │             │  │              │  │              │
        │ Launch      │  │ ECS Cluster  │  │ EKS Cluster  │
        │ Template    │  │      │       │  │      │       │
        │      │      │  │   Fargate    │  │ Node Groups  │
        │     ASG     │  │    Tasks     │  │      │       │
        │      │      │  │      │       │  │ Kubernetes   │
        │  User Data  │  │  Container   │  │      │       │
        │      │      │  │      │       │  │     Pods     │
        │   Docker    │  │   Docker     │  │ Deployment   │
        │      │      │  │      │       │  │ Service      │
        │   Flask     │  │    Flask     │  │ Ingress      │
        └──────┬──────┘  └──────┬───────┘  │    Flask     │
               │                │          └──────┬───────┘
               │                │                 │
               └────────────────┼─────────────────┘
                                │
                  ┌─────────────┴─────────────┐
                  │      COMMON AWS           │
                  │      INFRASTRUCTURE       │
                  └─────────────┬─────────────┘
                                │
          ┌─────────────────────┼─────────────────────────┐
          │                     │                         │
          ▼                     ▼                         ▼
    ┌───────────┐        ┌──────────────┐          ┌──────────┐
    │    RDS    │        │   Secrets    │          │    S3    │
    │   MySQL   │        │   Manager    │          │          │
    └─────┬─────┘        └──────────────┘          └──────────┘
          │
          │
    ┌─────┴─────────────────────────────────┐
    │                 VPC                   │
    │                                       │
    │  Subnets │ Route Tables │ NAT │ SGs  │
    └───────────────────────────────────────┘


    ┌──────────┐       ┌──────────────┐       ┌───────────┐
    │   ECR    │       │  CloudWatch  │       │    SNS    │
    │  Images  │       │ Logs/Metrics │       │  Alerts   │
    └──────────┘       └──────────────┘       └───────────┘


                              IAM
                               │
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
                EC2           ECS           EKS
```

---

# Application Architecture

The application is a Flask-based web application with database connectivity and order-processing functionality.

```text
application/
│
├── app.py
├── orders.py
├── db.py
├── index.html
├── requirements.txt
└── Dockerfile
```

### Application Components

| File               | Purpose                                     |
| ------------------ | ------------------------------------------- |
| `app.py`           | Flask application and HTTP routes           |
| `orders.py`        | Order-processing/application logic          |
| `db.py`            | Database connection and database operations |
| `index.html`       | Application frontend                        |
| `requirements.txt` | Python dependencies                         |
| `Dockerfile`       | Container image definition                  |

The application is packaged into a Docker image and stored in **Amazon ECR**.

---

# AWS Infrastructure

The infrastructure is managed using **Terraform**.

The project separates infrastructure into three major areas:

```text
terraform/
│
├── common/
├── ec2/
├── ecs/
└── eks/
```

The `common` infrastructure is shared by all deployment models.

---

# Common Infrastructure

```text
terraform/common/
│
├── vpc.tf
├── subnets.tf
├── routing.tf
├── security-groups.tf
├── iam.tf
├── rds.tf
├── secrets.tf
├── s3.tf
├── ecr.tf
├── alb.tf
├── cloudwatch.tf
└── sns.tf
```

### VPC

Provides the networking foundation:

* VPC
* Public subnets
* Private application subnets
* Private database subnets
* Internet Gateway
* NAT Gateway
* Route tables
* Security groups

---

### RDS

Amazon RDS for MySQL provides the application's relational database.

The application connects to RDS from the private network.

```text
Application
     │
     │ MySQL 3306
     ▼
   RDS MySQL
```

Database access is restricted using security groups.

---

### Secrets Manager

Database credentials and other sensitive configuration are stored in **AWS Secrets Manager** rather than hard-coded inside the application.

```text
EC2 ───────┐
           │
ECS ───────┼──► Secrets Manager
           │
EKS ───────┘
```

IAM permissions control which workloads can retrieve secrets.

---

### S3

Amazon S3 provides object storage for application files and other required assets.

---

### ECR

Amazon Elastic Container Registry stores the Docker image used by ECS and EKS.

```text
Dockerfile
    │
    ▼
Docker Build
    │
    ▼
Docker Image
    │
    ▼
   ECR
   │ │
   │ └──────────► ECS/Fargate
   │
   └────────────► EKS
```

---

### Application Load Balancer

The ALB provides the common entry point for HTTP traffic.

```text
Internet
   │
   ▼
 ALB
   │
   ├──► EC2
   │
   ├──► ECS/Fargate
   │
   └──► EKS
```

The ALB can perform health checks and route traffic to healthy application targets.

---

### CloudWatch

CloudWatch provides:

* Application logs
* Container logs
* EC2 monitoring
* ECS monitoring
* EKS-related monitoring
* Metrics
* Alarms
* Operational troubleshooting data

---

### SNS

Amazon SNS is used for operational alerts.

```text
CloudWatch Alarm
       │
       ▼
      SNS
       │
       ▼
 Notification
```

---

# EC2 Deployment

```text
terraform/ec2/
│
├── launch-template.tf
├── autoscaling.tf
├── user-data.tf
└── ssm.tf
```

The EC2 deployment uses:

```text
ALB
 │
 ▼
Auto Scaling Group
 │
 ├── EC2 Instance
 │     │
 │     ├── Docker
 │     └── Flask
 │
 └── EC2 Instance
       │
       ├── Docker
       └── Flask
```

### Launch Template

Defines the EC2 configuration, including:

* AMI
* Instance type
* IAM instance profile
* Security group
* User data
* Application configuration

### Auto Scaling Group

Provides:

* Multiple EC2 instances
* Instance replacement
* Scaling
* High availability across subnets/AZs

### User Data

Bootstraps the EC2 instance and installs/configures the required software.

Typical flow:

```text
EC2 starts
   │
   ▼
User Data
   │
   ├── Install Docker
   ├── Configure services
   ├── Authenticate/pull image
   └── Start application
```

### SSM

AWS Systems Manager provides operational access to EC2 without requiring direct SSH access.

---

# ECS/Fargate Deployment

```text
terraform/ecs/
│
├── cluster.tf
├── fargate.tf
├── task-definition.tf
├── service.tf
├── task-role.tf
└── execution-role.tf
```

Architecture:

```text
ALB
 │
 ▼
ECS Service
 │
 ▼
Fargate Tasks
 │
 └── Flask Container
```

### ECS Cluster

Provides the logical cluster where ECS services run.

### Fargate

Runs containers without managing EC2 worker nodes.

### Task Definition

Defines:

* Container image
* CPU
* Memory
* Container port
* Environment variables
* Secrets
* Logging
* IAM roles

### ECS Service

Maintains the desired number of running tasks and integrates the service with the ALB.

### IAM Roles

Two separate roles are used:

**Execution Role**

Used by ECS to perform actions such as:

* Pulling images from ECR
* Sending logs to CloudWatch
* Retrieving required resources during startup

**Task Role**

Used by the running application container to access AWS services.

Example:

```text
Flask Container
      │
      ▼
   Task Role
      │
      ├── Secrets Manager
      ├── S3
      └── Other AWS APIs
```

---

# EKS Deployment

```text
terraform/eks/
│
├── cluster.tf
├── node-groups.tf
├── kubernetes.tf
├── deployment.tf
├── service.tf
├── ingress.tf
├── configmap.tf
├── secret.tf
├── probes.tf
└── hpa.tf
```

Architecture:

```text
ALB
 │
 ▼
Ingress
 │
 ▼
Kubernetes Service
 │
 ▼
Deployment
 │
 ├── Pod
 │    └── Flask Container
 │
 ├── Pod
 │    └── Flask Container
 │
 └── Pod
      └── Flask Container
```

### EKS Cluster

Provides the managed Kubernetes control plane.

### Node Groups

Provide EC2 worker nodes where Kubernetes workloads run.

### Deployment

Defines the desired application state, including:

* Number of replicas
* Container image
* Container port
* Resource configuration
* Environment configuration

### Service

Provides stable networking to the application pods.

### Ingress

Provides external HTTP/HTTPS routing into the Kubernetes application.

### ConfigMap

Stores non-sensitive configuration.

### Secret

Stores sensitive Kubernetes configuration.

### Probes

The application uses Kubernetes health checks such as:

* Liveness probe
* Readiness probe

Example:

```text
Kubernetes
    │
    ▼
Readiness Probe
    │
    ├── Healthy ──► Receive traffic
    │
    └── Unhealthy ──► Remove from service
```

### HPA

Horizontal Pod Autoscaler can automatically adjust the number of application pods based on resource utilization.

---

# Repository Structure

```text
.
├── application/
│   ├── app.py
│   ├── orders.py
│   ├── db.py
│   ├── index.html
│   ├── requirements.txt
│   └── Dockerfile
│
└── terraform/
    │
    ├── common/
    │   ├── vpc.tf
    │   ├── subnets.tf
    │   ├── routing.tf
    │   ├── security-groups.tf
    │   ├── iam.tf
    │   ├── rds.tf
    │   ├── secrets.tf
    │   ├── s3.tf
    │   ├── ecr.tf
    │   ├── alb.tf
    │   ├── cloudwatch.tf
    │   └── sns.tf
    │
    ├── ec2/
    │   ├── launch-template.tf
    │   ├── autoscaling.tf
    │   ├── user-data.tf
    │   └── ssm.tf
    │
    ├── ecs/
    │   ├── cluster.tf
    │   ├── fargate.tf
    │   ├── task-definition.tf
    │   ├── service.tf
    │   ├── task-role.tf
    │   └── execution-role.tf
    │
    └── eks/
        ├── cluster.tf
        ├── node-groups.tf
        ├── kubernetes.tf
        ├── deployment.tf
        ├── service.tf
        ├── ingress.tf
        ├── configmap.tf
        ├── secret.tf
        ├── probes.tf
        └── hpa.tf
```

---

# Traffic Flow

The production traffic flow is:

```text
User
 │
 ▼
Internet
 │
 ▼
Application Load Balancer
 │
 ├─────────────────┐
 │                 │
 ▼                 ▼
EC2             ECS/Fargate
 │                 │
 │                 │
 └────────┬────────┘
          │
          ▼
       Flask
          │
          ▼
      RDS MySQL
```

For Kubernetes:

```text
User
 │
 ▼
Internet
 │
 ▼
ALB
 │
 ▼
Ingress
 │
 ▼
Kubernetes Service
 │
 ▼
Flask Pods
 │
 ▼
RDS MySQL
```

---

# Security Architecture

The project follows a private-by-default architecture.

```text
                    INTERNET
                       │
                       ▼
                      ALB
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
             EC2      ECS      EKS
              │        │        │
              └────────┼────────┘
                       │
                       ▼
                     RDS
```

Security groups restrict communication between application components.

Examples:

```text
ALB SG
  │
  └──► Application SG : 8000

Application SG
  │
  └──► Database SG : 3306
```

Database access is not exposed directly to the internet.

Secrets are stored in AWS Secrets Manager rather than committed to Git.

---

# Technology Stack

### Application

* Python
* Flask
* Gunicorn
* MySQL
* Docker

### AWS

* Amazon VPC
* Amazon EC2
* Auto Scaling
* Application Load Balancer
* Amazon ECS
* AWS Fargate
* Amazon EKS
* Amazon RDS MySQL
* Amazon S3
* Amazon ECR
* AWS Secrets Manager
* Amazon CloudWatch
* Amazon SNS
* AWS Systems Manager
* AWS IAM

### Infrastructure as Code

* Terraform

### Container Orchestration

* ECS/Fargate
* Kubernetes/EKS

---

# Deployment Model

The project demonstrates three different ways of running the same application.

```text
                 SAME FLASK APPLICATION
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
            EC2          ECS          EKS
             │            │            │
             ▼            ▼            ▼
           Docker       Docker       Docker
             │            │            │
             └────────────┼────────────┘
                          │
                          ▼
                  COMMON AWS SERVICES
```

This makes it possible to compare:

| Platform    | Compute Model         | Orchestration      |
| ----------- | --------------------- | ------------------ |
| EC2         | Virtual machines      | Auto Scaling Group |
| ECS/Fargate | Serverless containers | ECS                |
| EKS         | Kubernetes containers | Kubernetes         |

---

# Infrastructure as Code

Terraform is used to provision and manage the AWS environment.

The infrastructure is separated into:

```text
common/
    Shared AWS infrastructure

ec2/
    EC2 deployment

ecs/
    ECS/Fargate deployment

eks/
    EKS/Kubernetes deployment
```

This separation allows the same application to be deployed using different compute platforms while reusing the common infrastructure.

---

# Operational Monitoring

The environment is designed for production-style troubleshooting.

CloudWatch is used to monitor:

```text
EC2
 │
 ├── CPU
 ├── Instance health
 └── Application logs

ECS
 │
 ├── Task health
 ├── Container logs
 └── Service status

EKS
 │
 ├── Pod health
 ├── Container logs
 └── Kubernetes workload status

Application
 │
 ├── /health
 └── /db
```

CloudWatch alarms can trigger SNS notifications for operational incidents.

---

# Production Troubleshooting Flow

When the application is unavailable, troubleshooting follows the request path:

```text
1. Internet
      │
      ▼
2. ALB
      │
      ▼
3. Target Health
      │
      ▼
4. EC2 / ECS / EKS
      │
      ▼
5. Container
      │
      ▼
6. Flask
      │
      ▼
7. Database Connection
      │
      ▼
8. RDS
```

Typical checks include:

```text
ALB
 ├── Listener
 ├── Target group
 └── Target health

Compute
 ├── Instance/task/pod status
 ├── CPU
 ├── Memory
 └── Application status

Application
 ├── Flask/Gunicorn
 ├── /health
 ├── /db
 └── Application logs

Database
 ├── Security group
 ├── Connectivity
 ├── Credentials
 └── RDS status
```

---

# Goals of the Project

This project demonstrates practical experience with:

* Building a Flask application
* Containerizing applications with Docker
* Infrastructure as Code with Terraform
* AWS networking
* EC2 Auto Scaling
* ECS/Fargate
* EKS/Kubernetes
* Application Load Balancing
* RDS MySQL
* Secrets Manager
* S3
* ECR
* IAM
* CloudWatch
* SNS
* Systems Manager
* Production troubleshooting
* Running the same application across multiple compute platforms

---

# Project Objective

The main objective is to build and operate a **production-style AWS environment** where the same Flask application can be deployed using:

```text
EC2 + ASG
     │
     ├── Docker
     └── Flask

ECS + Fargate
     │
     ├── ECS Service
     └── Flask Container

EKS + Kubernetes
     │
     ├── Deployment
     ├── Service
     ├── Ingress
     ├── Probes
     └── HPA
```

All three deployment models integrate with shared AWS services including:

```text
VPC
RDS
Secrets Manager
S3
ECR
ALB
CloudWatch
SNS
IAM
```

This repository is intended as a hands-on demonstration of **AWS Cloud Support, DevOps, Infrastructure as Code, containerization, and Kubernetes operations**.

