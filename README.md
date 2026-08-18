AWS EC2 CI/CD Web Server

A hands-on cloud computing project that demonstrates how to deploy and automatically update a web application on Amazon EC2 using GitHub Actions, AWS IAM, OpenID Connect (OIDC), and AWS Systems Manager.

Every change pushed to the main branch triggers an automated deployment to the EC2 web server.

⸻

Project Overview

This project started as a simple static website hosted on an Amazon EC2 instance and was later extended into an automated CI/CD deployment pipeline.

The deployment process eliminates the need to manually SSH into the server whenever the website is updated.

The final workflow is:

Developer
    │
    │ git push
    ▼
GitHub Repository
    │
    │ push to main
    ▼
GitHub Actions
    │
    │ OIDC authentication
    ▼
AWS IAM Role
    │
    │ AssumeRoleWithWebIdentity
    ▼
AWS Systems Manager
    │
    │ RunShellScript
    ▼
Amazon EC2
    │
    │ git fetch / git reset
    ▼
Web Server
    │
    │ /var/www/html/index.html
    ▼
Live Website

⸻

Objectives

The main objectives of this project were to:

* Deploy a web server on Amazon EC2.
* Host a static website on the EC2 instance.
* Manage the project source code using Git and GitHub.
* Create an automated CI/CD pipeline using GitHub Actions.
* Authenticate GitHub Actions with AWS using OIDC.
* Use AWS IAM to control deployment permissions.
* Use AWS Systems Manager to execute deployment commands on EC2.
* Automatically synchronize the EC2 server with the GitHub repository.
* Troubleshoot real AWS, IAM, SSM, Git, and CI/CD failures.

⸻

Technologies Used

Technology	Purpose
Amazon EC2	Hosts the web server
Amazon Linux 2023	Operating system for the EC2 instance
Git	Version control
GitHub	Source code repository
GitHub Actions	CI/CD automation
AWS IAM	Identity and access management
AWS IAM OIDC	Authentication between GitHub Actions and AWS
AWS STS	Provides temporary credentials through role assumption
AWS Systems Manager	Executes deployment commands on EC2
HTML	Website content

⸻

Architecture

The application uses GitHub as the source of truth and AWS Systems Manager as the deployment mechanism.

Deployment Architecture

                         ┌──────────────────┐
                         │    Developer     │
                         │                  │
                         │  Modify website  │
                         └────────┬─────────┘
                                  │
                              git push
                                  │
                                  ▼
                         ┌──────────────────┐
                         │     GitHub       │
                         │   Repository     │
                         └────────┬─────────┘
                                  │
                           Push to main
                                  │
                                  ▼
                         ┌──────────────────┐
                         │ GitHub Actions   │
                         │                  │
                         │ Deploy Website   │
                         └────────┬─────────┘
                                  │
                            OIDC Token
                                  │
                                  ▼
                         ┌──────────────────┐
                         │     AWS IAM      │
                         │                  │
                         │ GitHubActions-   │
                         │ EC2-Deploy       │
                         └────────┬─────────┘
                                  │
                        AssumeRoleWithWebIdentity
                                  │
                                  ▼
                         ┌──────────────────┐
                         │ AWS Systems      │
                         │ Manager (SSM)    │
                         └────────┬─────────┘
                                  │
                            Run commands
                                  │
                                  ▼
                         ┌──────────────────┐
                         │   Amazon EC2     │
                         │                  │
                         │ Amazon Linux 2023│
                         │                  │
                         │ Web Server       │
                         └────────┬─────────┘
                                  │
                                  ▼
                         /var/www/html/
                                  │
                                  ▼
                            Live Website

A dedicated architecture diagram is also planned under docs/.

⸻

CI/CD Workflow

The deployment is triggered whenever code is pushed to the main branch.

1. Developer modifies the website

The website source code is stored in:

index.html

2. Changes are committed

git add .
git commit -m "Update website content"

3. Changes are pushed to GitHub

git push origin main

4. GitHub Actions starts automatically

The workflow is configured to run on pushes to main:

on:
  push:
    branches:
      - main

5. GitHub authenticates with AWS

The workflow uses GitHub’s OIDC identity provider instead of storing long-lived AWS access keys.

permissions:
  id-token: write
  contents: read

GitHub obtains an OIDC token and uses it to assume the AWS IAM deployment role.

6. AWS IAM authorizes the deployment

The IAM role allows GitHub Actions to authenticate with AWS and perform the required deployment operations.

7. AWS Systems Manager sends commands to EC2

GitHub Actions uses:

AWS-RunShellScript

to execute commands on the EC2 instance.

8. EC2 synchronizes with GitHub

The deployment process runs:

git fetch origin
git reset --hard origin/main

This ensures the EC2 repository matches the latest version of the main branch.

9. Website files are deployed

The latest HTML file is copied into the web server directory:

sudo cp /home/ec2-user/cloud-ec2-web-server/index.html /var/www/html/index.html

10. The live website is updated

The web server now serves the latest version of the website.

⸻

GitHub Actions Workflow

The deployment workflow is located at:

.github/workflows/deploy.yml

The workflow performs the following operations:

1. Authenticates with AWS using OIDC.
2. Assumes the deployment IAM role.
3. Sends an SSM command to the EC2 instance.
4. Fetches the latest GitHub changes.
5. Resets the EC2 repository to origin/main.
6. Copies the updated website to the web server directory.
7. Waits for the SSM command to complete.
8. Reports deployment success or failure.

⸻

Security Design

A key security decision in this project was avoiding long-lived AWS access keys in GitHub.

Instead, the project uses GitHub Actions OIDC authentication.

GitHub Actions
      │
      │ OIDC
      ▼
AWS STS
      │
      │ AssumeRoleWithWebIdentity
      ▼
AWS IAM Role
      │
      ▼
AWS Systems Manager
      │
      ▼
EC2

Benefits

* No AWS access keys stored in the repository.
* GitHub receives temporary AWS credentials.
* IAM controls what the deployment workflow can access.
* The IAM trust policy restricts which GitHub repository and branch can assume the role.
* GitHub Actions only receives the permissions required by the workflow.

⸻

Project Structure

cloud-ec2-web-server/
│
├── index.html
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── docs/
│   ├── architecture.md
│   ├── deployment.md
│   └── troubleshooting.md
│
└── README.md

⸻

Deployment

Prerequisites

The following are required:

* AWS account
* Running EC2 instance
* Amazon Linux 2023
* Git installed on EC2
* Web server installed and configured
* AWS Systems Manager Agent
* EC2 IAM instance profile
* GitHub repository
* GitHub Actions
* GitHub OIDC provider configured in AWS
* IAM deployment role
* AWS Systems Manager permissions

Manual deployment test

The deployment can be tested manually on the EC2 instance:

cd /home/ec2-user/cloud-ec2-web-server
git fetch origin
git reset --hard origin/main
sudo cp index.html /var/www/html/index.html

Automated deployment

After the initial configuration, deployment only requires:

git add .
git commit -m "Update website"
git push origin main

GitHub Actions handles the remaining deployment process automatically.

⸻

Troubleshooting

During development, several real-world issues were encountered and resolved.

SSM Agent could not connect to Systems Manager

Error:

no EC2 instance role found

Cause:

The EC2 instance did not have the required IAM instance profile.

Resolution:

An EC2 IAM role was configured and attached to the instance so the SSM Agent could communicate with AWS Systems Manager.

⸻

GitHub Actions could not assume the AWS role

Error:

Not authorized to perform sts:AssumeRoleWithWebIdentity

Cause:

The GitHub Actions OIDC identity did not satisfy the IAM role’s trust policy.

Resolution:

The GitHub OIDC provider and IAM trust policy were configured to allow the appropriate GitHub repository and branch to assume the role.

⸻

SSM GetCommandInvocation permission denied

Error:

not authorized to perform: ssm:GetCommandInvocation

Cause:

The GitHub Actions deployment role could send commands but could not retrieve their execution status.

Resolution:

The required Systems Manager permission was added to the deployment role.

⸻

Git reported dubious repository ownership

Error:

fatal: detected dubious ownership in repository

Cause:

The SSM execution environment was not running Git as the repository owner.

Resolution:

Git operations were changed to execute as ec2-user:

sudo -u ec2-user git

⸻

Git reported $HOME not set

Error:

fatal: $HOME not set

Cause:

The SSM execution environment did not provide the expected home directory environment variable.

Resolution:

The deployment was changed to avoid global Git configuration and instead use Git’s repository-specific configuration:

git -c safe.directory=/home/ec2-user/cloud-ec2-web-server

⸻

Lessons Learned

This project provided hands-on experience with several important cloud engineering concepts.

AWS EC2

Learned how to:

* Launch an EC2 instance.
* Connect using SSH.
* Configure a Linux server.
* Host a website.
* Manage files and permissions.

IAM

Learned how:

* IAM roles work.
* Trust policies control who can assume roles.
* Identity policies control what actions a role can perform.
* EC2 instances use instance profiles.
* GitHub Actions can authenticate without long-lived credentials.

OIDC

Learned how GitHub Actions can establish a trusted identity with AWS through OpenID Connect.

AWS Systems Manager

Learned how to:

* Register an EC2 instance with SSM.
* Send commands remotely.
* Retrieve command execution status.
* Automate server operations without relying on SSH.

GitHub Actions

Learned how to:

* Create workflow files.
* Trigger workflows on Git pushes.
* Configure AWS authentication.
* Automate deployments.
* Debug failed CI/CD pipelines.

Troubleshooting

One of the most valuable parts of the project was debugging failures instead of simply rebuilding the environment.

The project required troubleshooting:

* IAM permissions
* OIDC trust relationships
* EC2 instance profiles
* Systems Manager
* Git repository ownership
* Linux users and permissions
* Git synchronization
* GitHub Actions workflow failures

⸻

Future Improvements

Potential improvements for future versions include:

* Add HTTPS using a domain name and SSL/TLS.
* Add Amazon CloudWatch monitoring.
* Add health checks.
* Add automated tests before deployment.
* Add deployment rollback capabilities.
* Use Infrastructure as Code with Terraform or AWS CloudFormation.
* Move the EC2 instance into a more secure VPC architecture.
* Add an Application Load Balancer.
* Store configuration in AWS Systems Manager Parameter Store.
* Implement blue/green or rolling deployments.
* Replace the static website with a full application.
* Add a database layer using Amazon RDS.

⸻

Project Status

Status: Completed

The project successfully demonstrates automated deployment from GitHub to an Amazon EC2 web server using:

GitHub
   ↓
GitHub Actions
   ↓
OIDC
   ↓
AWS IAM
   ↓
AWS Systems Manager
   ↓
Amazon EC2
   ↓
Web Server

⸻

Author

Allen Halog

Hands-on AWS and Cloud Computing Learning Project.
