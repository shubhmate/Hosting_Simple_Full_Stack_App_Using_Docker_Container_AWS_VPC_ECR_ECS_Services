# Hosting a Simple Full-Stack App with Docker + AWS (VPC, ECR, ECS)

A short, simple, step-by-step guide to containerize a frontend and a backend, push images to AWS ECR, and run them on AWS ECS (Fargate) inside a VPC.

Summary of key choices in this README:
- Frontend listens on port 3000
- Backend listens on port 5000
- VPC: minimum 2 public subnets (frontend) and 2 private subnets (backend)
- NAT Gateway: optional (recommended if private subnets need outbound access)
- ALB (Application Load Balancer): optional (recommended for production/HTTPS)

---

## Table of contents
- Overview
- What’s in this repo
- Quick start (local)
- Docker Compose example
- Build and push images to ECR
- AWS network (VPC) and recommended layout
- ECS (Fargate) deployment essentials
- Task definition example (minimal)
- Security groups and health checks
- Environment variables
- Troubleshooting
- Infrastructure as code & CI/CD notes

---

## Overview
This repo shows how to:
1. Build Docker images for frontend and backend.
2. Push those images to AWS ECR.
3. Run them on AWS ECS (Fargate) in a VPC with at least 2 public and 2 private subnets.

Keep things simple:
- Frontend: port 3000 (exposed to users)
- Backend: port 5000 (ideally in private subnets)

NAT gateway and ALB are optional: you can run a small app without them during testing, but they are recommended for production.

---

## What’s in this repo
(Adjust if your repo differs)
- server/        — backend code (listening on port 5000)
- client/        — frontend code (listening on port 3000)
- Dockerfile(s)  — for frontend and backend
- docker-compose.yml (optional for local)
- infra/         — optional scripts / Terraform / CloudFormation
- README.md

---

## Quick start (local)
1. Build backend:
   - docker build -t myapp-backend:local ./server
2. Run backend:
   - docker run -e PORT=5000 -p 5000:5000 myapp-backend:local
3. Build frontend:
   - docker build -t myapp-frontend:local ./client
4. Run frontend:
   - docker run -e REACT_APP_API_URL=http://localhost:5000 -p 3000:3000 myapp-frontend:local
5. Open: http://localhost:3000

---

## Docker Compose example
Place this as docker-compose.yml (adjust build paths and env as needed).

version: "3.8"
services:
  backend:
    build: ./server
    image: myapp-backend:local
    environment:
      - PORT=5000
    ports:
      - "5000:5000"
  frontend:
    build: ./client
    image: myapp-frontend:local
    environment:
      - REACT_APP_API_URL=http://localhost:5000
    ports:
      - "3000:3000"

Start locally:
- docker-compose build
- docker-compose up

---

## Build and push images to AWS ECR (example)
Replace placeholders: <aws-region>, <account-id>, <repo-name>.

1. Create ECR repos (one-time):
   - aws ecr create-repository --repository-name myapp-backend --region <aws-region>
   - aws ecr create-repository --repository-name myapp-frontend --region <aws-region>

2. Authenticate Docker to ECR:
   - aws ecr get-login-password --region <aws-region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<aws-region>.amazonaws.com

3. Build and tag:
   - docker build -t myapp-backend:latest ./server
   - docker tag myapp-backend:latest <account-id>.dkr.ecr.<aws-region>.amazonaws.com/myapp-backend:latest
   - docker build -t myapp-frontend:latest ./client
   - docker tag myapp-frontend:latest <account-id>.dkr.ecr.<aws-region>.amazonaws.com/myapp-frontend:latest

4. Push:
   - docker push <account-id>.dkr.ecr.<aws-region>.amazonaws.com/myapp-backend:latest
   - docker push <account-id>.dkr.ecr.<aws-region>.amazonaws.com/myapp-frontend:latest

---

## AWS network (VPC) — recommended minimal layout
For production or realistic testing, a VPC with separated subnets is recommended.

Minimum recommended setup:
- VPC (CIDR e.g., 10.0.0.0/16)
- 2 Public subnets (spread across AZs) — run frontend tasks (if they need public access)
- 2 Private subnets (spread across AZs) — run backend tasks and databases
- NAT Gateway: optional but recommended if tasks in private subnets need outbound internet access (e.g., to download packages, connect to external APIs). You can skip NAT gateway if backend tasks do not require outbound internet.
- Internet Gateway attached to VPC for public subnet access (if ALB or public tasks are used)

Notes:
- Using 2 subnets in different Availability Zones improves availability.
- If you choose to put frontend in public subnets, set public IP assignment in ECS service or front with ALB in public subnets.
- ALB is optional: you can expose the frontend directly (less secure/robust). For HTTPS and routing, use ALB.

---

## ECS (Fargate) deployment essentials
1. IAM:
   - Create an ECS Task Execution Role (AmazonECSTaskExecutionRolePolicy).
   - Create a task role if the app needs AWS access.

2. ECS Cluster:
   - Create a Fargate cluster.

3. Task Definitions:
   - Frontend container: image from ECR, container port 3000
   - Backend container: image from ECR, container port 5000
   - Configure logging (awslogs) and environment variables/secrets

4. Services:
   - Create a service for frontend and backend (Fargate).
   - Set desired counts (start with 1, increase for redundancy).
   - Choose subnets:
     - Frontend service in public subnets (or private with ALB, depending on design).
     - Backend service in private subnets.
   - Set security groups (see next section).

5. Optional ALB:
   - Add ALB in public subnets, attach listener (80/443), target frontend service (port 3000).
   - Configure listener rules and health checks.

---

## Task definition example (minimal)
A minimal container entry for the backend (replace image URIs):

{
  "family": "myapp-backend",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "backend",
      "image": "<account-id>.dkr.ecr.<aws-region>.amazonaws.com/myapp-backend:latest",
      "portMappings": [{ "containerPort": 5000, "protocol": "tcp" }],
      "environment": [
        { "name": "PORT", "value": "5000" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp-backend",
          "awslogs-region": "<aws-region>",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}

Change the frontend container's portMappings.containerPort to 3000 and REACT_APP_API_URL accordingly.

---

## Security groups & health checks (suggested)
Security groups:
- ALB (if used) SG:
  - Inbound: 80 (or 443) from 0.0.0.0/0
  - Outbound: allow to target group/backend SG as needed
- Frontend service SG:
  - If public: allow inbound from ALB SG on port 3000 (or allow 0.0.0.0/0 if no ALB and public)
  - Outbound: allow to backend SG on port 5000
- Backend service SG:
  - Inbound: allow from frontend SG on port 5000
  - Outbound: allow to database or internet (via NAT gateway) as needed

Health checks:
- Frontend: health check path, e.g., / or /health on port 3000
- Backend: set health check, e.g., /health on port 5000

---

## Environment variables (example)
Frontend (.env or encoded into task definition):
- REACT_APP_API_URL=https://<your-api-domain>  (frontend expects backend at 5000)

Backend:
- PORT=5000
- DATABASE_URL=postgres://user:pass@db-host:5432/dbname
- JWT_SECRET=changeme
- NODE_ENV=production

Do not commit secrets to source control. Use AWS Secrets Manager or SSM Parameter Store and reference in task definitions.

---

## Troubleshooting (quick)
- Docker push to ECR failed:
  - Confirm AWS ECR get-login-password output and repo exists.
- ECS tasks stuck in PENDING:
  - Check chosen subnets (must be valid for awsvpc), ENI limits, and whether there are enough IPs available.
  - If backend is in private subnets and needs outbound internet, ensure a NAT gateway exists (or use VPC endpoints).
- 502 / 503 from ALB:
  - Ensure the container listens on the configured containerPort (frontend 3000, backend 5000).
  - Ensure health check path returns 200.
- Empty logs:
  - Confirm awslogs config and that the task execution role has CloudWatch Logs permissions.

---

## Infrastructure as code & CI/CD (notes)
- Use Terraform or CloudFormation to create reliably:
  - VPC, subnets, NAT gateways (optional), route tables
  - ECR repositories, IAM roles, ECS cluster, task definitions, services, ALB
- CI/CD (GitHub Actions):
  - Build images on push, run tests, push to ECR, refresh the ECS service, or create a new deployment.

Example GitHub Actions steps:
- Checkout, set up Docker buildx, login to ECR, build and push images, then update ECS service with AWS CLI or CodeDeploy.

---
