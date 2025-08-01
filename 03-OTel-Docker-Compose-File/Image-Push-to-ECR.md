# Push Docker Images to Amazon ECR

This guide explains how to push Docker images from your local machine to Amazon Elastic Container Registry (ECR)

## Prerequisites

Before you begin, ensure you have the following installed and configured:

- [AWS CLI](https://aws.amazon.com/cli/)
- [Docker](https://www.docker.com/get-started)
- Appropriate AWS IAM permissions for ECR (e.g., `AmazonEC2ContainerRegistryFullAccess`)
- An AWS account and your AWS Account ID

---

## Installation & Setup

#### 1. Authenticate Docker to Amazon ECR
```
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
```
#### 2. Create an ECR Repository
```
aws ecr create-repository --repository-name <your-repo-name> --region <region>
```

#### 3. Tag Your Docker Image
```
docker tag <your-local-image>:<tag> <aws_account_id>.dkr.ecr.<region>.amazonaws.com/<your-repo-name>:<tag>
```

#### 4. Push the Image to ECR
```
docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/<your-repo-name>:<tag>

```

#### Example
**Here's a full example with sample values:**
```
# Authenticate
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp --region us-east-1

# Tag image
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Push image
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

```
### Parameters Explanation

| Parameter            | Description                         |
| -------------------- | ----------------------------------- |
| `<region>`           | Your AWS region (e.g., `us-east-1`) |
| `<aws_account_id>`   | Your 12-digit AWS account ID        |
| `<your-repo-name>`   | Name of your ECR repository         |
| `<your-local-image>` | Name of the local Docker image      |
| `<tag>`              | Image tag (e.g., `latest`, `v1.0`)  |

### Common Issues and Troubleshooting

**1. Authentication Failure**

- Ensure AWS credentials are configured correctly
- Verify IAM permissions for ECR

**2. Push Failed**
- Ensure the repository exists
- Make sure the image tag and repository name match
- Check that the Docker daemon is running

**3. Common Error Messages**

- Error Message	Solution
- Repository not found	Create the repository first
- No basic auth credentials	Re-authenticate using get-login-password
- Access denied	Check your IAM permissions

