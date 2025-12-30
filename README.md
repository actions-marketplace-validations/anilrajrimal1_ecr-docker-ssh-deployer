# Deploy Docker Image GitHub Action

> Deploy Docker images from **Amazon ECR** to a remote server over **SSH** using **Docker Compose**, powered by **GitHub Actions OIDC** (no AWS access keys), with environment secrets fetched from **Phase**.

[![GitHub release (latest by date)](https://img.shields.io/github/v/release/anilrajrimal1/ecr-docker-ssh-deployer)](https://github.com/anilrajrimal1/ecr-docker-ssh-deployer/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

### Description
This GitHub Action deploys Docker images to a remote server using **SSH** and **Docker Compose**.
It authenticates to AWS using **GitHub Actions OpenID Connect (OIDC)**, logs into **Amazon ECR**, pulls images on the **remote Docker daemon**, and deploys them using a branch-specific Docker Compose file. Secrets and environment variables are dynamically fetched from **Phase**, making this action ideal for **staging / production deployments** without storing long-lived credentials.


### Prerequisites

Before using this action, ensure:

- You have an **AWS IAM Role** configured for GitHub OIDC
- The role has permission to **pull images from ECR**
- Your GitHub workflow includes:
  ```yaml
  permissions:
    id-token: write
    contents: read
  ```
- Your deployment server:
    - Has Docker & Docker Compose installed
    - Is reachable via SSH
    - Allows key-based SSH authentication

### AWS OIDC Setup (One Time)
1. Create an OIDC provider in AWS IAM:
   - URL: https://token.actions.githubusercontent.com
   - Audience: sts.amazonaws.com

2. Create an IAM role with trust policy:
```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:*"
    }
  }
}

```
3. Attach an IAM policy allowing `ecr:GetAuthorizationToken` and image pull actions.

### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `SSH_PRIVATE_KEY` | ✅ | SSH private key to connect to the deployment server |
| `AWS_ROLE_ARN` | ✅ | IAM Role ARN to assume via GitHub OIDC |
| `AWS_REGION` | ✅ | AWS region for ECR |
| `ECR_REPOSITORY` | ✅ | ECR repository name |
| `DEPLOYMENT_SERVER_IP` | ✅ | IP address of the deployment target server |
| `DEPLOYMENT_SERVER_USERNAME` | ✅ | SSH username (default: `ubuntu`) |
| `PHASE_SERVICE_TOKEN` | ✅ | Phase Service Token |
| `PHASE_APP_ID` | ✅ | Unique identifier for the app in Phase |
| `BRANCH_NAME` | ✅ | Git branch name (used for env + compose file) |
| `PHASE_HOST` | ❌ | Phase host URL, only for self-hosted environments |


### How It Works

1. Sets up SSH access using ssh-agent

2. Assumes AWS IAM role via OIDC

3. Logs in to Amazon ECR

4. Fetches environment variables from Phase

5. Copies branch-specific Docker Compose file
`docker-compose.<branch>.yml`
6. Uses DOCKER_HOST=ssh://... to:
   - docker compose pull
   - docker compose up -d


### Usage Example

```yaml
name: Deploy to Server

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy
        uses: anilrajrimal1/ecr-docker-ssh-deployer@v1.0.2
        with:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          AWS_ROLE_ARN: ${{ vars.AWS_ROLE_ARN }}
          AWS_REGION: ${{ vars.AWS_REGION }}
          ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
          DEPLOYMENT_SERVER_IP: ${{ vars.DEPLOYMENT_SERVER_IP }}
          DEPLOYMENT_SERVER_USERNAME: ${{ vars.DEPLOYMENT_SERVER_USERNAME }}
          BRANCH_NAME: ${{ github.ref_name }}
          PHASE_SERVICE_TOKEN: ${{ secrets.PHASE_SERVICE_TOKEN }}
          PHASE_APP_ID: ${{ vars.PHASE_APP_ID }}

```


### Notes

- This action uses my [Phase Secrets Fetch Action](https://github.com/anilrajrimal1/phase-secrets-fetch-action) to fetch `.env` files dynamically.
- SSH must be key-based (no password support)
- Docker authentication happens on the remote daemon
- No AWS credentials are stored on the server
