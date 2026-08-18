Deployment Guide

This document describes how the AWS EC2 CI/CD web server was configured and how website changes are automatically deployed from GitHub to Amazon EC2.

⸻

1. Architecture Overview

The deployment pipeline consists of the following components:

Developer
    |
    | git push
    v
GitHub Repository
    |
    | Push to main
    v
GitHub Actions
    |
    | OIDC authentication
    v
AWS IAM Role
    |
    | Temporary AWS credentials
    v
AWS Systems Manager
    |
    | RunShellScript
    v
Amazon EC2
    |
    | Git synchronization
    v
/var/www/html/index.html
    |
    v
Live Website

The architecture diagram is available at:

docs/architecture.png

⸻

2. EC2 Configuration

The website is hosted on an Amazon EC2 instance running Amazon Linux 2023.

The EC2 instance contains:

/home/ec2-user/cloud-ec2-web-server/

This directory contains the Git repository used to synchronize the server with GitHub.

The web server serves the website from:

/var/www/html/

The deployment process copies the latest index.html from the Git repository into this directory.

⸻

3. Git Repository

The project source code is hosted on GitHub.

The repository contains:

cloud-ec2-web-server/
│
├── index.html
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── docs/
│   ├── architecture.png
│   ├── architecture.md
│   ├── deployment.md
│   └── troubleshooting.md
│
└── README.md

Git is used to track all changes to the project.

⸻

4. GitHub Actions

The automated deployment workflow is located at:

.github/workflows/deploy.yml

The workflow is triggered whenever a commit is pushed to the main branch.

on:
  push:
    branches:
      - main

This means that a normal Git workflow automatically starts the deployment:

git add .
git commit -m "Update website"
git push origin main

⸻

5. GitHub OIDC Authentication

The workflow uses GitHub Actions OpenID Connect to authenticate with AWS.

The workflow requests an OIDC identity token:

permissions:
  id-token: write
  contents: read

The AWS credentials action then uses this identity to assume the deployment IAM role:

- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ vars.AWS_ROLE_ARN }}
    aws-region: ap-southeast-1

No long-lived AWS access key or secret key is stored in the GitHub repository.

⸻

6. AWS IAM Deployment Role

A dedicated IAM role was created for GitHub Actions:

GitHubActions-EC2-Deploy

The role has two important components:

Trust Policy

The trust policy allows GitHub Actions to assume the role using OIDC.

The trust relationship verifies:

* The GitHub OIDC provider.
* The OIDC audience.
* The authorized GitHub repository.
* The authorized branch.

This prevents unrelated GitHub repositories from assuming the deployment role.

Permissions Policy

The role is granted the AWS permissions required by the deployment workflow.

The deployment requires Systems Manager operations such as:

ssm:SendCommand
ssm:GetCommandInvocation

The principle of least privilege should be applied when expanding this policy.

⸻

7. EC2 IAM Role

The EC2 instance also requires an IAM role.

This role allows the SSM Agent running on EC2 to communicate with AWS Systems Manager.

The relationship is:

EC2
 |
 | IAM Instance Profile
 v
EC2 IAM Role
 |
 v
AWS Systems Manager

This is separate from the GitHub Actions deployment role.

There are therefore two different IAM roles involved:

GitHub Actions
      |
      v
GitHubActions-EC2-Deploy
      |
      | Controls what GitHub Actions can do
      v
AWS Systems Manager
EC2
 |
 v
EC2 Instance Role
 |
 | Allows EC2 to communicate with SSM
 v
AWS Systems Manager

⸻

8. AWS Systems Manager

AWS Systems Manager is used to execute deployment commands on the EC2 instance.

GitHub Actions sends an AWS-RunShellScript command using:

ssm:SendCommand

The deployment command performs the following operations:

1. Navigate to the project directory.
2. Fetch the latest GitHub repository state.
3. Reset the local repository to origin/main.
4. Copy the updated website to the web server directory.

The important synchronization commands are:

git fetch origin
git reset --hard origin/main

This ensures the EC2 repository matches the latest version of the GitHub main branch.

⸻

9. Deployment Command

The deployment workflow executes commands similar to:

set -e
cd /home/ec2-user/cloud-ec2-web-server
sudo -u ec2-user git -c safe.directory=/home/ec2-user/cloud-ec2-web-server fetch origin
sudo -u ec2-user git -c safe.directory=/home/ec2-user/cloud-ec2-web-server reset --hard origin/main
sudo cp /home/ec2-user/cloud-ec2-web-server/index.html /var/www/html/index.html

set -e

Stops the deployment if a command fails.

This prevents later deployment steps from executing after an earlier command has failed.

git fetch origin

Downloads the latest information from GitHub.

git reset --hard origin/main

Makes the EC2 repository match the remote main branch.

sudo cp

Copies the latest website into the directory served by the web server.

⸻

10. Deployment Status Monitoring

The workflow does not simply send the SSM command and assume it succeeded.

It retrieves the command status using:

ssm:GetCommandInvocation

The workflow checks for states such as:

Success
Failed
Cancelled
TimedOut

If the command succeeds, the workflow reports:

Deployment completed successfully.

If the command fails, the workflow displays the SSM output and exits with an error.

This makes deployment failures easier to diagnose.

⸻

11. Complete Deployment Flow

A normal deployment follows this sequence:

1. Developer modifies index.html
              |
              v
2. git add .
              |
              v
3. git commit
              |
              v
4. git push origin main
              |
              v
5. GitHub detects push
              |
              v
6. GitHub Actions starts
              |
              v
7. GitHub requests OIDC token
              |
              v
8. AWS STS validates the token
              |
              v
9. AWS IAM role is assumed
              |
              v
10. GitHub Actions calls SSM
              |
              v
11. EC2 receives deployment command
              |
              v
12. EC2 fetches latest GitHub changes
              |
              v
13. EC2 resets to origin/main
              |
              v
14. index.html copied to /var/www/html
              |
              v
15. Website serves the new version

⸻

12. Testing Automatic Deployment

To test the CI/CD pipeline:

Modify the website

Edit:

index.html

For example:

<h1>CI/CD Deployment Test</h1>

Commit the change

git add index.html
git commit -m "Test automatic deployment"

Push the change

git push origin main

Monitor GitHub Actions

Open the repository on GitHub and navigate to:

Actions

Select:

Deploy Website

The workflow should show a successful deployment.

Verify the EC2 repository

SSH into the instance:

ssh -i <your-key>.pem ec2-user@<EC2-PUBLIC-IP>

Then:

cd /home/ec2-user/cloud-ec2-web-server
git log -1 --oneline

The latest commit should match the GitHub main branch.

Verify the deployed file

cat /var/www/html/index.html

The file should contain the latest website content.

⸻

13. Deployment Result

After successful deployment:

GitHub
   |
   | Push
   v
GitHub Actions
   |
   | OIDC
   v
AWS IAM
   |
   | SSM
   v
EC2
   |
   | Update files
   v
Web Server
   |
   v
Updated Website

The result is an automated deployment pipeline where a Git push can update the live EC2-hosted website without manually SSHing into the server for every deployment.

⸻

14. Future Improvements

The current deployment pipeline can be extended with:

* Automated tests before deployment.
* Deployment approval environments.
* Rollback to previous versions.
* Application health checks.
* CloudWatch monitoring.
* HTTPS and a custom domain.
* Infrastructure as Code.
* Load balancing.
* Auto Scaling.
* Blue/green deployments.
* Containerization with Docker.
* Deployment to Amazon ECS or EKS.
