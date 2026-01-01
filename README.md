🏦 Blue-Green Deployment for Banking Website using AWS & Jenkins
📌 Project Overview

This project demonstrates a Blue–Green Deployment strategy implemented for a banking website using AWS Cloud services and Jenkins CI/CD pipeline.
The main objective is to achieve zero-downtime deployments, ensuring continuous availability of the application during updates.

The application is hosted on Amazon EC2 instances, traffic is managed using an Application Load Balancer (ALB), and build artifacts are stored in Amazon S3. Jenkins automates the entire deployment process.

🚀 Key Features

✅ Zero-downtime deployment using Blue–Green strategy

✅ Automated CI/CD pipeline using Jenkins

✅ Artifact storage and versioning using Amazon S3

✅ Traffic switching using Application Load Balancer

✅ Easy rollback in case of deployment failure

✅ Secure access using IAM Roles & Security Groups

🏗️ Architecture Overview
🔵 Blue Environment

Current live production environment

Serves all user traffic initially

🟢 Green Environment

New version of the application is deployed here

Tested independently before traffic switch

🔁 Deployment Cycle

Deploy new code to Green

Perform health checks

Switch traffic Blue → Green

Keep Blue as rollback backup

🧰 Technologies & AWS Services Used
☁️ AWS Services

Amazon EC2 – Hosts Blue, Green, and Jenkins servers

Application Load Balancer (ALB) – Routes traffic between environments

Target Groups – Separate routing for Blue & Green

Amazon S3 – Stores build artifacts (index-xx.html)

IAM – Secure access for Jenkins and EC2 instances

VPC & Security Groups – Network isolation and access control

⚙️ DevOps Tools

Jenkins – CI/CD automation

GitHub – Source code management

AWS CLI – Infrastructure automation
 🔄 CI/CD Workflow
Developer pushes code → GitHub
        ↓
GitHub Webhook triggers Jenkins
        ↓
Jenkins builds artifact
        ↓
Artifact stored in Amazon S3
        ↓
Artifact deployed to GREEN EC2
        ↓
Health check on GREEN
        ↓
ALB switches traffic (BLUE → GREEN)

📦 Artifact Management (Amazon S3)

Each build generates a versioned artifact:

index-42.html
index-43.html
index-44.html


Artifacts are stored in:

s3://banking-bluegreen-artifacts/


EC2 instances pull artifacts directly from S3 during deployment

🔁 Traffic Switching Logic

ALB listens on HTTP port 80

Initially routes traffic to Blue Target Group

On successful deployment:

Jenkins updates ALB listener to forward traffic to Green Target Group

On failure:

Traffic automatically rolls back to Blue

🛡️ Security Implementation

SSH access controlled via Security Groups

Jenkins uses IAM Role to access S3 and ALB

No hardcoded AWS credentials in pipeline

Principle of least privilege followed

📁 Repository Structure
banking-blue-green/
│
├── index.html          # Banking website
├── Jenkinsfile         # Jenkins CI/CD pipeline
├── README.md           # Project documentation

📊 Result

Application updates deployed without downtime

Users experience uninterrupted service

Faster and safer deployments compared to traditional methods

Immediate rollback available in case of errors

🎯 Conclusion

This project successfully demonstrates how Blue–Green deployment can be implemented using AWS and Jenkins to achieve high availability, reliability, and automation for real-world applications like banking systems.

📚 References

https://docs.aws.amazon.com/elasticloadbalancing

https://docs.aws.amazon.com/ec2

https://docs.aws.amazon.com/s3

https://www.jenkins.io/doc

https://aws.amazon.com/devops/