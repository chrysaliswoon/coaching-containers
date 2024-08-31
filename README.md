# Activity Brief

Propose a process that will automate the building of your container image and deployment of your container onto a container platform. Have a think of what are the needed components, where they should exist, and how it will be solutioned. Sketch it out if needed.

## Requirements:
- Container build and deploy is to be driven by Github workflows
- Workflows can be triggered manually or upon merge to main
- Can consist of single or multiple workflows
- Store image on own AWS ECR
- Deploy to own AWS ECS on a provided network
- Application to be exposed on port 9090


### Components Required:

1. **GitHub Repository**: Contains your application code and the GitHub Actions workflows.
2. **GitHub Actions**: For building the container image, pushing it to AWS ECR, and deploying it to AWS ECS.
3. **AWS ECR (Elastic Container Registry)**: To store the Docker container images.
4. **AWS ECS (Elastic Container Service)**: To manage and deploy your containerized application.
5. **AWS IAM Roles**: For permissions and access control.
6. **AWS VPC (Virtual Private Cloud)**: Networking configuration for ECS.

### Workflow Overview:

1. **Build and Push Image Workflow**:
   - Triggered manually or automatically upon merging to the `main` branch.
   - Builds the Docker image.
   - Pushes the Docker image to AWS ECR.

2. **Deploy Workflow**:
   - Triggered manually or automatically upon successful build and push.
   - Updates the ECS service to use the new image.

### Account Set-up

**Part 1: Set up AWS Credentials for Github Repo**

This purpose of this step is for your Github workflow to authenticate with the AWS account on your behalf to perform actions (such as creating resources)

1. Go to your Github repository page and go to Settings
2. Go to Secrets and Variables -> Actions
3. Click “New Repository Secret”
4. You will have to create 2 secrets with the following name:
    - **Name**: AWS_ACCESS_KEY_ID
    - **Value**: Key in your Access Key ID value
    - **Name**: AWS_SECRET_ACCESS_KEY
    - **Value**: Key in your Secret Access Key ID value


### Detailed Steps:

#### 1. **GitHub Actions Workflow for Building and Pushing Docker Image**

- **File Location**: `.github/workflows/build-and-push.yml`

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.ECR_REPOSITORY }} .
          docker tag ${{ secrets.ECR_REPOSITORY }}:latest ${{ secrets.ECR_URI }}:latest
          docker push ${{ secrets.ECR_URI }}:latest

    env:
      AWS_REGION: ${{ secrets.AWS_REGION }}
```

#### 2. **GitHub Actions Workflow for Deploying to ECS**

- **File Location**: `.github/workflows/deploy-to-ecs.yml`

```yaml
name: Deploy to ECS

on:
  workflow_run:
    workflows: ["Build and Push Docker Image"]
    types:
      - completed

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up AWS CLI
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Update ECS service
        run: |
          aws ecs update-service --cluster ${{ secrets.ECS_CLUSTER }} --service ${{ secrets.ECS_SERVICE }} --force-new-deployment

    env:
      AWS_REGION: ${{ secrets.AWS_REGION }}
```

### Secrets Configuration

Ensure you have the following secrets configured in your GitHub repository settings:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`
- `ECR_REPOSITORY` (The name of your ECR repository)
- `ECR_URI` (The URI of your ECR repository, e.g., `123456789012.dkr.ecr.us-west-2.amazonaws.com/my-app`)
- `ECS_CLUSTER` (The name of your ECS cluster)
- `ECS_SERVICE` (The name of your ECS service)

### Network and Port Configuration

- Ensure your ECS service is correctly configured to use port 9090. This should be set in your ECS task definition.
- Verify that the security group associated with your ECS service allows inbound traffic on port 9090.

### Summary

1. **GitHub Actions Workflow for Build and Push**: This workflow builds your Docker image and pushes it to AWS ECR.
2. **GitHub Actions Workflow for Deploy**: This workflow deploys the new image to your ECS service.
3. **Secrets Management**: Store sensitive information like AWS credentials and repository details as GitHub secrets.
4. **ECS and Networking**: Ensure ECS is configured with the correct port and network settings.

This setup ensures a streamlined and automated CI/CD pipeline for deploying your containerized application to AWS ECS.





To visualize the CI/CD workflow for building and deploying your container image using GitHub Actions, here's a simplified diagram:

### Visual Workflow Diagram

```
GitHub Repository
   ├── .github/workflows/
   │     ├── build-and-push.yml
   │     └── deploy-to-ecs.yml
   ├── Dockerfile
   └── Application Code
       ↓
       ┌───────────────┐
       │  GitHub       │
       │  Workflow     │
       │  Triggered    │
       └───────────────┘
            │
            ▼
 ┌─────────────────────────┐
 │ Build and Push Image   │
 │ Workflow                │
 │ (build-and-push.yml)    │
 └─────────────────────────┘
            │
            ▼
 ┌─────────────────────────┐
 │ Amazon ECR              │
 │ (Store Docker Image)    │
 └─────────────────────────┘
            │
            ▼
 ┌─────────────────────────┐
 │ Deploy to ECS           │
 │ Workflow                │
 │ (deploy-to-ecs.yml)     │
 └─────────────────────────┘
            │
            ▼
 ┌─────────────────────────┐
 │ Amazon ECS              │
 │ (Deploy and Run         │
 │  Docker Container)      │
 └─────────────────────────┘
            │
            ▼
 ┌─────────────────────────┐
 │ Application Exposed on  │
 │ Port 9090               │
 └─────────────────────────┘
```
