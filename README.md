# Full-Stack Deployment: Flask & Express on AWS ECS

This project demonstrates how to containerize and deploy a dual-container application (Flask Backend & Express Frontend) using **Amazon ECR**, **Amazon ECS (Fargate)**, and **Amazon VPC**.

## 🏗️ Architecture Detail
- **Frontend**: Node.js / Express.js (Dockerised)
- **Backend**: Python / Flask (Dockerised)
- **Registry**: Amazon ECR (Elastic Container Registry)
- **Orchestration**: Amazon ECS (Elastic Container Service)
- **Infrastructure**: AWS Fargate (Serverless Compute)
- **Network**: Amazon VPC (Subnets(Public & Private), Security Groups & Internet Gateway)

## 🛠️ Prerequisites

- AWS Account with an IAM user having `AdministratorAccess`.
- AWS CLI installed and configured (`aws configure`).
- Docker Desktop installed.
- Python 3.x and Node.js installed locally (for testing).

## 🛠️ Setup & Deployment

### 1. Containerize & Push to ECR
First, build your images and push them to your private ECR repositories:

```bash
# Login to ECR
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com

# Build and Push Backend
docker build -t flask-backend ./backend
docker tag flask-backend:latest <account_id>.dkr.ecr.<region>://
docker push <account_id>.dkr.ecr.<region>://

# Build and Push Frontend
docker build -t express-frontend ./frontend
docker tag express-frontend:latest <account_id>.dkr.ecr.<region>://
docker push <account_id>.dkr.ecr.<region>://
```

## 2. Network Configuration (VPC)
*   **Create VPC**: Set up a VPC with Public Subnets.
*   **Security Group (Frontend)**: Allow inbound HTTP (Port 3000) from `0.0.0.0/0`.
*   **Security Group (Backend)**: Allow inbound traffic on Port 5000 (Note: Internal traffic via localhost typically bypasses external SG rules within the same task).

## 3. ECS Deployment (Fargate)
*   **Task Definition**: Create a new Fargate task definition. Add both the Flask and Express containers using their respective ECR URIs.
*   **Cluster**: Create an ECS Cluster (Networking only / Fargate).
*   **Service**: Launch a service inside your cluster using the Task Definition.
    *   Set **Desired Tasks** to 1.
    *   Ensure **Assign Public IP** is set to **ENABLED**.

## 🔌 How It Works: Container Communication
This project uses **Shared Task Networking** (awsvpc mode). Since both containers run within the same ECS Task, they share the same network namespace.

*   **Communication Flow**:
    *   **External Traffic**: User hits the Public IP on Port 3000 (Express Frontend).
    *   **Internal Traffic**: Express communicates with Flask via `http://localhost:5000`.
*   **Key Advantage**: Containers talk to each other directly over `localhost` without needing external routing or Service Discovery.
*   **Security**: The Backend is shielded from direct public internet access, accepting traffic only via the shared task environment.

## 💰 Cost Management & Cleanup
*   **Monitoring**: A **CloudWatch Billing Alarm** is recommended (e.g., $5 threshold).
*   **Teardown Order**:
    1.  **ECS Service**: Update desired tasks to 0 and delete.
    2.  **ECS Cluster**: Delete the cluster.
    3.  **Task Definitions**: Deregister definitions.
    4.  **ECR**: Delete images/repositories.
    5.  **VPC**: Delete NAT Gateways (if any) and release Elastic IPs.
