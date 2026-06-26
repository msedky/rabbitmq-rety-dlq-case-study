# AWS Deployment Guide — RabbitMQ Retry and DLQ Sample

This document covers deploying the `rabbitmq-rety-dlq-sample` project to AWS.

The local setup (Docker Compose) is documented in [README.md](./README.md).

---

## AWS Architecture

```
GitHub Actions CI/CD
        │
        ├── build & push payment-service  ──► ECR
        ├── build & push invoice-service  ──► ECR
        └── deploy ──────────────────────────► ECS Fargate
                                                    │
                                       ┌────────────┴────────────┐
                                       │                         │
                               payment-service           invoice-service
                               (ECS Fargate Task)        (ECS Fargate Task)
                                       │                         │
                              ┌────────┴──────┐        ┌────────┴──────┐
                              │               │        │               │
                           RDS            Amazon MQ  RDS          Amazon MQ
                        payment_db       (RabbitMQ) invoice_db   (RabbitMQ)
                                               │
                                    (shared broker instance)

                         ALB (Application Load Balancer)
                              │               │
                     /payment/*        /invoice/*
                              │               │
                     payment-service   invoice-service
```

## AWS Services Used

| Service | Purpose |
|---|---|
| **ECR** | Container registry for Docker images |
| **ECS Fargate** | Serverless container runtime |
| **RDS PostgreSQL** | Managed databases for payment_db and invoice_db |
| **Amazon MQ** | Managed RabbitMQ broker |
| **ALB** | Application Load Balancer — routes traffic to each service |
| **Secrets Manager** | Stores RDS and Amazon MQ credentials securely |
| **IAM** | Task execution roles with least-privilege permissions |
| **VPC** | Network isolation — services in private subnets |
| **GitHub Actions** | CI/CD pipeline — build, push to ECR, deploy to ECS |

---

## Prerequisites

- AWS account with sufficient permissions (or AdministratorAccess for initial setup)
- AWS CLI installed and configured

```bash
aws configure
# Enter: AWS Access Key ID, Secret Access Key, region (e.g. eu-west-1), output format (json)
```

- Docker installed locally
- GitHub repository with the project pushed

---

## Phase 1 — ECR: Container Registry

### 1.1 Create ECR Repositories

```bash
aws ecr create-repository \
  --repository-name rabbitmq-retry-dlq/payment-service \
  --region eu-west-1

aws ecr create-repository \
  --repository-name rabbitmq-retry-dlq/invoice-service \
  --region eu-west-1
```

Note the `repositoryUri` from each output — you will need it in the GitHub Actions workflow.

### 1.2 Authenticate Docker to ECR

```bash
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin \
  <your-account-id>.dkr.ecr.eu-west-1.amazonaws.com
```

### 1.3 Build and Push Images (manual first push)

```bash
# payment-service
docker build -t rabbitmq-retry-dlq/payment-service:latest ./payment-service
docker tag rabbitmq-retry-dlq/payment-service:latest \
  <your-account-id>.dkr.ecr.eu-west-1.amazonaws.com/rabbitmq-retry-dlq/payment-service:latest
docker push \
  <your-account-id>.dkr.ecr.eu-west-1.amazonaws.com/rabbitmq-retry-dlq/payment-service:latest

# invoice-service
docker build -t rabbitmq-retry-dlq/invoice-service:latest ./invoice-service
docker tag rabbitmq-retry-dlq/invoice-service:latest \
  <your-account-id>.dkr.ecr.eu-west-1.amazonaws.com/rabbitmq-retry-dlq/invoice-service:latest
docker push \
  <your-account-id>.dkr.ecr.eu-west-1.amazonaws.com/rabbitmq-retry-dlq/invoice-service:latest
```

---

## Phase 2 — RDS: Managed PostgreSQL

Create two RDS instances — one per service database.

### 2.1 Create payment_db

```bash
aws rds create-db-instance \
  --db-instance-identifier payment-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16 \
  --master-username postgres \
  --master-user-password <your-password> \
  --db-name payment_db \
  --allocated-storage 20 \
  --no-publicly-accessible \
  --region eu-west-1
```

### 2.2 Create invoice_db

```bash
aws rds create-db-instance \
  --db-instance-identifier invoice-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16 \
  --master-username postgres \
  --master-user-password <your-password> \
  --db-name invoice_db \
  --allocated-storage 20 \
  --no-publicly-accessible \
  --region eu-west-1
```

### 2.3 Note the Endpoints

Once the instances are available (takes ~5 minutes), retrieve the endpoints:

```bash
aws rds describe-db-instances \
  --db-instance-identifier payment-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text

aws rds describe-db-instances \
  --db-instance-identifier invoice-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text
```

### 2.4 Security Group

Allow port 5432 inbound only from the ECS tasks security group — not from the internet.

```bash
# Get your default VPC ID
aws ec2 describe-vpcs --query 'Vpcs[?IsDefault==`true`].VpcId' --output text

# Create a security group for RDS
aws ec2 create-security-group \
  --group-name rds-retry-dlq-sg \
  --description "RDS access for rabbitmq-retry-dlq ECS tasks"

# Add inbound rule — replace <ecs-sg-id> after creating the ECS security group in Phase 4
aws ec2 authorize-security-group-ingress \
  --group-name rds-retry-dlq-sg \
  --protocol tcp \
  --port 5432 \
  --source-group <ecs-tasks-sg-id>
```

---

## Phase 3 — Amazon MQ: Managed RabbitMQ

### 3.1 Create the Broker

```bash
aws mq create-broker \
  --broker-name rabbitmq-retry-dlq-broker \
  --engine-type RABBITMQ \
  --engine-version 3.13 \
  --deployment-mode SINGLE_INSTANCE \
  --host-instance-type mq.t3.micro \
  --publicly-accessible false \
  --user '[{"username":"mquser","password":"<your-mq-password>","groups":["administrator"]}]' \
  --region eu-west-1
```

> `SINGLE_INSTANCE` is sufficient for a portfolio project and is the most cost-effective option.

### 3.2 Retrieve the Broker Endpoints

```bash
aws mq describe-broker \
  --broker-id <broker-id> \
  --query 'BrokerInstances[0]' \
  --output json
```

Note two endpoints:
- **AMQP endpoint** (port 5671, TLS) — used by the Spring Boot services
- **Console URL** (port 443) — RabbitMQ Management UI

### 3.3 SSL Configuration Note

Amazon MQ uses TLS on port 5671 by default. Add the following to your `application.properties` or pass as environment variables in ECS:

```properties
spring.rabbitmq.port=5671
spring.rabbitmq.ssl.enabled=true
```

Or as ECS environment variables:

```
SPRING_RABBITMQ_PORT=5671
SPRING_RABBITMQ_SSL_ENABLED=true
```

### 3.4 Security Group

Allow port 5671 inbound only from the ECS tasks security group:

```bash
aws ec2 create-security-group \
  --group-name mq-retry-dlq-sg \
  --description "Amazon MQ access for rabbitmq-retry-dlq ECS tasks"

aws ec2 authorize-security-group-ingress \
  --group-name mq-retry-dlq-sg \
  --protocol tcp \
  --port 5671 \
  --source-group <ecs-tasks-sg-id>
```

---

## Phase 4 — Secrets Manager: Secure Credentials

Store sensitive values in AWS Secrets Manager instead of plain environment variables.

### 4.1 Store RDS Passwords

```bash
aws secretsmanager create-secret \
  --name rabbitmq-retry-dlq/payment-db-password \
  --secret-string '{"password":"<your-rds-password>"}' \
  --region eu-west-1

aws secretsmanager create-secret \
  --name rabbitmq-retry-dlq/invoice-db-password \
  --secret-string '{"password":"<your-rds-password>"}' \
  --region eu-west-1
```

### 4.2 Store Amazon MQ Credentials

```bash
aws secretsmanager create-secret \
  --name rabbitmq-retry-dlq/mq-credentials \
  --secret-string '{"username":"mquser","password":"<your-mq-password>"}' \
  --region eu-west-1
```

### 4.3 IAM Policy for ECS Task Role

Create a policy that allows ECS tasks to read these secrets:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": [
        "arn:aws:secretsmanager:eu-west-1:<account-id>:secret:rabbitmq-retry-dlq/*"
      ]
    }
  ]
}
```

```bash
aws iam create-policy \
  --policy-name rabbitmq-retry-dlq-secrets-policy \
  --policy-document file://secrets-policy.json
```

Attach this policy to the ECS task execution role created in the next phase.

---

## Phase 5 — ECS Fargate: Container Runtime

### 5.1 Create the ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name rabbitmq-retry-dlq-cluster \
  --region eu-west-1
```

### 5.2 Create IAM Task Execution Role

```bash
aws iam create-role \
  --role-name ecsTaskExecutionRole-retry-dlq \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach the AWS managed policy for ECS task execution (ECR pull, CloudWatch logs)
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole-retry-dlq \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Attach the custom secrets policy from Phase 4
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole-retry-dlq \
  --policy-arn arn:aws:iam::<account-id>:policy/rabbitmq-retry-dlq-secrets-policy
```

### 5.3 Create Task Definition — payment-service

Save as `payment-service-task-def.json`:

```json
{
  "family": "payment-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole-retry-dlq",
  "taskRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole-retry-dlq",
  "containerDefinitions": [
    {
      "name": "payment-service",
      "image": "<account-id>.dkr.ecr.eu-west-1.amazonaws.com/rabbitmq-retry-dlq/payment-service:latest",
      "portMappings": [{"containerPort": 8081, "protocol": "tcp"}],
      "environment": [
        {"name": "SPRING_DATASOURCE_URL",
         "value": "jdbc:postgresql://<payment-rds-endpoint>:5432/payment_db"},
        {"name": "SPRING_DATASOURCE_USERNAME", "value": "postgres"},
        {"name": "SPRING_RABBITMQ_HOST", "value": "<amazonmq-amqp-endpoint>"},
        {"name": "SPRING_RABBITMQ_PORT", "value": "5671"},
        {"name": "SPRING_RABBITMQ_SSL_ENABLED", "value": "true"}
      ],
      "secrets": [
        {
          "name": "SPRING_DATASOURCE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:eu-west-1:<account-id>:secret:rabbitmq-retry-dlq/payment-db-password:password::"
        },
        {
          "name": "SPRING_RABBITMQ_USERNAME",
          "valueFrom": "arn:aws:secretsmanager:eu-west-1:<account-id>:secret:rabbitmq-retry-dlq/mq-credentials:username::"
        },
        {
          "name": "SPRING_RABBITMQ_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:eu-west-1:<account-id>:secret:rabbitmq-retry-dlq/mq-credentials:password::"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/rabbitmq-retry-dlq/payment-service",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

```bash
aws ecs register-task-definition \
  --cli-input-json file://payment-service-task-def.json \
  --region eu-west-1
```

### 5.4 Create Task Definition — invoice-service

Save as `invoice-service-task-def.json` — same structure as above with the following differences:

- `"family"`: `"invoice-service"`
- `"image"`: invoice-service ECR image URI
- `"containerPort"`: `8082`
- `SPRING_DATASOURCE_URL`: invoice RDS endpoint, `invoice_db`
- `SPRING_DATASOURCE_PASSWORD` secret: invoice-db-password
- Add invoice-specific retry environment variables:

```json
{"name": "SPRING_RABBITMQ_LISTENER_SIMPLE_DEFAULT_REQUEUE_REJECTED", "value": "false"},
{"name": "APP_RETRY_INVOICE_MAX_ATTEMPTS", "value": "3"},
{"name": "APP_RETRY_INVOICE_DELAY", "value": "1000"},
{"name": "APP_RETRY_INVOICE_MULTIPLIER", "value": "1.0"},
{"name": "APP_RETRY_INVOICE_MAX_DELAY", "value": "1000"}
```

```bash
aws ecs register-task-definition \
  --cli-input-json file://invoice-service-task-def.json \
  --region eu-west-1
```

### 5.5 Create ECS Security Group

```bash
aws ec2 create-security-group \
  --group-name ecs-retry-dlq-tasks-sg \
  --description "ECS tasks for rabbitmq-retry-dlq"

# Allow inbound from ALB only (add ALB sg after creating it in Phase 6)
aws ec2 authorize-security-group-ingress \
  --group-name ecs-retry-dlq-tasks-sg \
  --protocol tcp \
  --port 8081 \
  --source-group <alb-sg-id>

aws ec2 authorize-security-group-ingress \
  --group-name ecs-retry-dlq-tasks-sg \
  --protocol tcp \
  --port 8082 \
  --source-group <alb-sg-id>
```

### 5.6 Create ECS Services

```bash
# payment-service
aws ecs create-service \
  --cluster rabbitmq-retry-dlq-cluster \
  --service-name payment-service \
  --task-definition payment-service \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[<private-subnet-id>],
    securityGroups=[<ecs-tasks-sg-id>],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=<payment-tg-arn>,containerName=payment-service,containerPort=8081" \
  --region eu-west-1

# invoice-service
aws ecs create-service \
  --cluster rabbitmq-retry-dlq-cluster \
  --service-name invoice-service \
  --task-definition invoice-service \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[<private-subnet-id>],
    securityGroups=[<ecs-tasks-sg-id>],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=<invoice-tg-arn>,containerName=invoice-service,containerPort=8082" \
  --region eu-west-1
```

---

## Phase 6 — ALB: Application Load Balancer

### 6.1 Create the ALB

```bash
aws elbv2 create-load-balancer \
  --name rabbitmq-retry-dlq-alb \
  --subnets <public-subnet-id-1> <public-subnet-id-2> \
  --security-groups <alb-sg-id> \
  --scheme internet-facing \
  --type application \
  --region eu-west-1
```

### 6.2 Create Target Groups

```bash
# payment-service target group
aws elbv2 create-target-group \
  --name payment-service-tg \
  --protocol HTTP \
  --port 8081 \
  --vpc-id <vpc-id> \
  --target-type ip \
  --health-check-path /actuator/health \
  --region eu-west-1

# invoice-service target group
aws elbv2 create-target-group \
  --name invoice-service-tg \
  --protocol HTTP \
  --port 8082 \
  --vpc-id <vpc-id> \
  --target-type ip \
  --health-check-path /actuator/health \
  --region eu-west-1
```

> Note: Make sure Spring Boot Actuator is on the classpath. If not, use a valid REST endpoint for health checks (e.g. `GET /api/v1/valid/payments` returning 200).

### 6.3 Create Listener and Routing Rules

```bash
# Create listener on port 80
aws elbv2 create-listener \
  --load-balancer-arn <alb-arn> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=fixed-response,FixedResponseConfig='{MessageBody=Not Found,StatusCode=404}' \
  --region eu-west-1

# Route /api/v1/valid/payments* and /api/v1/malformed/payments* to payment-service
aws elbv2 create-rule \
  --listener-arn <listener-arn> \
  --priority 10 \
  --conditions Field=path-pattern,Values='/api/v1/valid/payments*' \
  --actions Type=forward,TargetGroupArn=<payment-tg-arn>

# Route /api/v1/invoices* to invoice-service
aws elbv2 create-rule \
  --listener-arn <listener-arn> \
  --priority 20 \
  --conditions Field=path-pattern,Values='/api/v1/invoices*' \
  --actions Type=forward,TargetGroupArn=<invoice-tg-arn>

# Route malformed payment endpoints to payment-service
aws elbv2 create-rule \
  --listener-arn <listener-arn> \
  --priority 30 \
  --conditions Field=path-pattern,Values='/api/v1/malformed/*' \
  --actions Type=forward,TargetGroupArn=<payment-tg-arn>
```

---

## Phase 7 — GitHub Actions: CI/CD Pipeline

Create `.github/workflows/deploy.yml` in your repository:

```yaml
name: Build and Deploy to AWS ECS

on:
  push:
    branches:
      - main

env:
  AWS_REGION: eu-west-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.eu-west-1.amazonaws.com
  ECS_CLUSTER: rabbitmq-retry-dlq-cluster

jobs:
  deploy:
    name: Build, Push, Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push payment-service
        run: |
          docker build -t $ECR_REGISTRY/rabbitmq-retry-dlq/payment-service:${{ github.sha }} ./payment-service
          docker push $ECR_REGISTRY/rabbitmq-retry-dlq/payment-service:${{ github.sha }}
          docker tag $ECR_REGISTRY/rabbitmq-retry-dlq/payment-service:${{ github.sha }} \
            $ECR_REGISTRY/rabbitmq-retry-dlq/payment-service:latest
          docker push $ECR_REGISTRY/rabbitmq-retry-dlq/payment-service:latest

      - name: Build and push invoice-service
        run: |
          docker build -t $ECR_REGISTRY/rabbitmq-retry-dlq/invoice-service:${{ github.sha }} ./invoice-service
          docker push $ECR_REGISTRY/rabbitmq-retry-dlq/invoice-service:${{ github.sha }}
          docker tag $ECR_REGISTRY/rabbitmq-retry-dlq/invoice-service:${{ github.sha }} \
            $ECR_REGISTRY/rabbitmq-retry-dlq/invoice-service:latest
          docker push $ECR_REGISTRY/rabbitmq-retry-dlq/invoice-service:latest

      - name: Deploy payment-service to ECS
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service payment-service \
            --force-new-deployment \
            --region $AWS_REGION

      - name: Deploy invoice-service to ECS
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service invoice-service \
            --force-new-deployment \
            --region $AWS_REGION
```

### GitHub Actions Secrets to Configure

Go to your repository → Settings → Secrets and variables → Actions, and add:

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID |

---

## Service URLs After Deployment

| Service | URL |
|---|---|
| payment-service | `http://<alb-dns>/api/v1/valid/payments` |
| payment-service (malformed) | `http://<alb-dns>/api/v1/malformed/payments/payment-completed` |
| invoice-service | `http://<alb-dns>/api/v1/invoices` |
| RabbitMQ Management UI | Available via Amazon MQ console URL (port 443) |

Retrieve the ALB DNS name:

```bash
aws elbv2 describe-load-balancers \
  --names rabbitmq-retry-dlq-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text
```

---

## CloudWatch Logs

Application logs are sent to CloudWatch automatically via the `awslogs` log driver configured in the task definitions.

View logs:

```bash
# payment-service logs
aws logs tail /ecs/rabbitmq-retry-dlq/payment-service --follow

# invoice-service logs
aws logs tail /ecs/rabbitmq-retry-dlq/invoice-service --follow
```

Or open the AWS Console → CloudWatch → Log groups → `/ecs/rabbitmq-retry-dlq/`.

This replaces the local `docker logs` workflow and gives you visibility into retry attempts, DLQ routing, and consumer behavior in a real cloud environment.

---

## Teardown

When you are done, remove all resources to avoid ongoing charges:

```bash
# Scale down ECS services first
aws ecs update-service --cluster rabbitmq-retry-dlq-cluster --service payment-service --desired-count 0
aws ecs update-service --cluster rabbitmq-retry-dlq-cluster --service invoice-service --desired-count 0

# Delete ECS services
aws ecs delete-service --cluster rabbitmq-retry-dlq-cluster --service payment-service
aws ecs delete-service --cluster rabbitmq-retry-dlq-cluster --service invoice-service

# Delete ECS cluster
aws ecs delete-cluster --cluster rabbitmq-retry-dlq-cluster

# Delete ALB and target groups
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>
aws elbv2 delete-target-group --target-group-arn <payment-tg-arn>
aws elbv2 delete-target-group --target-group-arn <invoice-tg-arn>

# Delete Amazon MQ broker
aws mq delete-broker --broker-id <broker-id>

# Delete RDS instances
aws rds delete-db-instance --db-instance-identifier payment-db --skip-final-snapshot
aws rds delete-db-instance --db-instance-identifier invoice-db --skip-final-snapshot

# Delete ECR repositories
aws ecr delete-repository --repository-name rabbitmq-retry-dlq/payment-service --force
aws ecr delete-repository --repository-name rabbitmq-retry-dlq/invoice-service --force

# Delete Secrets Manager secrets
aws secretsmanager delete-secret --secret-id rabbitmq-retry-dlq/payment-db-password
aws secretsmanager delete-secret --secret-id rabbitmq-retry-dlq/invoice-db-password
aws secretsmanager delete-secret --secret-id rabbitmq-retry-dlq/mq-credentials
```

> Always verify in the AWS Console that all resources are removed, especially RDS instances and Amazon MQ brokers, as they are the most expensive components.
