#Highly Available Web App on EC2 (ALB + Auto Scaling) — HTTPS Enabled

Goal: Deploy a simple web app like a real production service using AWS best practices: private instances, public load balancer, auto scaling, health checks, scaling policy, and HTTPS.

---

## What I built

A highly available frontend deployment where:

- Source code is stored in **S3** as a zip file (`react-auth-portal.zip`)
- **EC2 instances run in private subnets** across **2 Availability Zones**
- An **internet-facing Application Load Balancer (ALB)** runs in public subnets
- An **Auto Scaling Group (ASG)** maintains capacity and **replaces instances automatically**
- **Nginx serves the built static frontend on port 80 inside the VPC**
- The public entrypoint is **HTTPS (443)** via **ACM certificate attached to the ALB**
- **HTTP (80) is redirected to HTTPS (443)** at the ALB
- Optional: Custom domain via **Route 53 → ALB**

---

## Deliverables checklist

- Launch Template with user data to install and run app (Nginx)
- Auto Scaling Group across 2 AZs
- Application Load Balancer + Target Group + health checks
- CloudWatch scaling to scale out on CPU (basic)
- Test evidence: ALB DNS works over HTTPS + instance replacement test

---

## Architecture

Traffic flow:

Client Browser  
→ **ALB (Public Subnets, HTTPS 443)**  
→ Target Group (HTTP health checks)  
→ EC2 Instances (Private Subnets, **HTTP 80 from ALB only**)  
→ NAT Gateway (outbound access for apt/npm installs)  
→ S3 (download zip during bootstrapping)

---

## Prerequisites

- VPC with:
  - 2 public subnets across 2 AZs
  - 2 private subnets across 2 AZs
  - 1 NAT Gateway
- S3 bucket in the same region
- Zip file: `react-auth-portal.zip`
- Route 53 hosted zone + domain (recommended for HTTPS + custom DNS)

---

## Implementation steps (AWS Console)

### 1) Upload the app zip to S3

Upload the frontend zip to a known path, for example:

`s3://YOUR_BUCKET/react-auth-portal/react-auth-portal.zip`

Record:
- Bucket name
- Object key

---

### 2) IAM role for EC2 (least privilege)

Create:
- IAM Role trusted by **EC2**
- Instance Profile attached to the role

Permissions:
- `s3:GetObject` scoped to only the zip path

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ListBucket",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::nodejs-prod-bucket"
        },
        {
            "Sid": "ReadAnyObjectInBucket",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::nodejs-prod-bucket/react-auth-portal.zip"
        }
    ]
}
  


### 3) Security groups (ALB → EC2 only)

**ALB Security Group**
Inbound:
- TCP 80 from 0.0.0.0/0 (used only for redirect)
- TCP 443 from 0.0.0.0/0 (main entrypoint)
Outbound:
- TCP 80 to EC2 Security Group

**EC2 Security Group**
Inbound:
- TCP 80 from ALB Security Group only
Outbound:
- Allow outbound (for OS updates, npm install, S3 access via NAT)

---

### 4) Create Target Group

Target type: Instances  
Protocol: HTTP  
Port: 80  
Health check path: `/`  
Success codes: `200-399`

---

### 5) Launch Template + User Data (bootstrapping)

This user data installs Nginx + Node, downloads the zip from S3, builds the app, and serves it via Nginx.

#!/bin/bash
set -euxo pipefail

exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1

apt-get update -y
apt-get install -y unzip nginx nodejs npm curl

cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -q awscliv2.zip
./aws/install || true

mkdir -p /opt/auth-portal
cd /opt/auth-portal

aws s3 cp s3://nodejs-prod-bucket/react-auth-portal.zip .

rm -rf app
mkdir app
unzip -q react-auth-portal.zip -d app

cd /opt/auth-portal/app/react-auth-portal

# If your package.json is inside a nested folder, cd into it here first.
npm install
npm run build

# Change BUILD_DIR if your frontend outputs a different folder
BUILD_DIR="build"
if [ ! -d "$BUILD_DIR" ]; then
  BUILD_DIR="dist"
fi

rm -rf /var/www/html/*
cp -r ${BUILD_DIR}/. /var/www/html/

systemctl enable nginx
systemctl restart nginx
systemctl restart nginx
