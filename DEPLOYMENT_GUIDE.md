# 🚀 AWS 3-Tier Web Application Deployment Guide

This document explains the complete deployment process of the AWS 3-Tier Web Application Architecture.

---

# 📌 Architecture Overview

The application is deployed using a production-inspired three-tier architecture.

```
Users
   │
Internet
   │
Internet Gateway
   │
External Application Load Balancer
   │
──────────────────────────────────────────────
Public Subnet (AZ-A)      Public Subnet (AZ-B)
Web EC2                   Web EC2
(Nginx + React)           (Nginx + React)
Auto Scaling Group
──────────────────────────────────────────────
             │
      Internal ALB
             │
──────────────────────────────────────────────
Private Subnet (AZ-A)     Private Subnet (AZ-B)
App EC2                   App EC2
(Node.js + PM2)           (Node.js + PM2)
Auto Scaling Group
──────────────────────────────────────────────
             │
Amazon RDS MySQL
(DB Subnet Group)

Private instances access the internet through a NAT Gateway.
```

---

# Phase 1 – Create Networking

## Create VPC

- Create a custom VPC.
- Configure CIDR block.
- Enable DNS Hostnames.
- Enable DNS Resolution.

---

## Create Subnets

Create:

- Public Subnet A
- Public Subnet B
- Private App Subnet A
- Private App Subnet B
- Private DB Subnet A
- Private DB Subnet B

---

## Internet Gateway

- Create Internet Gateway.
- Attach it to the VPC.

---

## NAT Gateway

Create:

- NAT Gateway
- Elastic IP

Update private route tables:

```
0.0.0.0/0 → NAT Gateway
```

This allows private EC2 instances to:

- Install packages
- Download dependencies
- Access AWS services

without exposing them to the Internet.

---

## Route Tables

Configure:

### Public Route Table

```
0.0.0.0/0 → Internet Gateway
```

Associate with:

- Public Subnet A
- Public Subnet B

### Private Route Table

```
0.0.0.0/0 → NAT Gateway
```

Associate with:

- Private App Subnets
- Private DB Subnets

---

# Phase 2 – Security

Create Security Groups:

- External ALB SG
- Web Tier SG
- Internal ALB SG
- App Tier SG
- Database SG

Allow only required traffic between tiers.

Example:

```
Internet
↓

External ALB

↓

Web Tier

↓

Internal ALB

↓

App Tier

↓

Database
```

---

# Phase 3 – IAM & S3

Create:

- IAM Role
- Instance Profile

Attach permissions required for:

- Amazon S3
- AWS Systems Manager (optional)

Create S3 bucket.

Upload:

```
application-code/
```

including:

- App Tier
- Web Tier
- nginx.conf

---

# Phase 4 – Database

Create Amazon RDS MySQL.

Recommended configuration:

```
Engine:
MySQL

Public Access:
Disabled

Subnet Group:
Private DB Subnets

Security Group:
Database SG
```

Connect:

```bash
mysql -h <RDS-ENDPOINT> -u <USERNAME> -p
```

Create database:

```sql
CREATE DATABASE webappdb;
```

Use database:

```sql
USE webappdb;
```

Create table:

```sql
CREATE TABLE transactions(
id INT AUTO_INCREMENT,
amount DECIMAL(10,2),
description VARCHAR(100),
PRIMARY KEY(id)
);
```

Insert sample data:

```sql
INSERT INTO transactions
(amount,description)
VALUES
('400','groceries');
```

Verify:

```sql
SELECT * FROM transactions;
```

---

# Phase 5 – Application Tier

Launch App Tier EC2.

Install:

```bash
sudo yum update -y

sudo yum install mysql -y
```

Install Node Version Manager.

Install Node.js.

```bash
nvm install 16

nvm use 16
```

Install PM2.

```bash
npm install -g pm2
```

Download application:

```bash
aws s3 cp s3://<BUCKET>/application-code/app-tier/ app-tier --recursive
```

Install dependencies.

```bash
npm install
```

Update:

```
DbConfig.js
```

Example:

```javascript
module.exports = Object.freeze({
DB_HOST: "<RDS_ENDPOINT>",
DB_USER: "<DB_USERNAME>",
DB_PWD: "<DB_PASSWORD>",
DB_DATABASE: "webappdb"
});
```

Start backend.

```bash
pm2 start index.js
```

Verify:

```bash
pm2 list
```

Health check:

```bash
curl http://localhost:4000/health
```

---

# Phase 6 – Internal Load Balancer

Create:

- Target Group
- Internal ALB

Configure health check.

Update:

```
nginx.conf
```

Replace:

```
proxy_pass http://<INTERNAL_ALB_DNS>;
```

Upload updated configuration to S3.

---

# Phase 7 – Web Tier

Launch Web Tier EC2.

Install Node.js.

Download code.

```bash
aws s3 cp s3://<BUCKET>/application-code/web-tier/ web-tier --recursive
```

Install dependencies.

```bash
npm install
```

Create React production build.

```bash
npm run build
```

Install Nginx.

```bash
sudo yum install nginx -y
```

Replace nginx configuration.

Restart service.

```bash
sudo systemctl restart nginx
```

Enable:

```bash
sudo systemctl enable nginx
```

---

# Phase 8 – External Load Balancer

Create:

- Target Group
- External ALB

Register Web Tier instances.

Verify:

Healthy Targets

---

# Phase 9 – Auto Scaling

Create Launch Template:

Web Tier

Create Launch Template:

App Tier

Create:

Web Tier Auto Scaling Group

Configuration:

```
Minimum:
2

Desired:
2

Maximum:
4
```

Create:

App Tier Auto Scaling Group

Configuration:

```
Minimum:
2

Desired:
2

Maximum:
4
```

Deploy instances across:

- Availability Zone A
- Availability Zone B

Verify:

- Healthy Instances
- Healthy Target Groups

---

# Phase 10 – Validation

Verify:

✅ Web Application

✅ Database Connectivity

✅ Target Group Health

✅ Internal ALB

✅ External ALB

✅ Auto Scaling Groups

✅ Launch Templates

✅ NAT Gateway

✅ Route Tables

✅ Security Groups

---

# Troubleshooting

Problems encountered during deployment:

- Node.js version incompatibility
- React build failures
- ESLint build issues
- PM2 startup configuration
- Nginx reverse proxy configuration
- ALB health check failures
- RDS connectivity issues
- Security Group configuration
- IAM role permissions
- SSM connectivity

Each issue was resolved through systematic debugging using logs, health checks, and AWS networking configuration.

---

# Cost Optimization

To minimize AWS costs:

- Used t3.micro instances.
- Used a single NAT Gateway.
- Used a single RDS instance.
- Enabled Auto Scaling only for demonstration.
- Did not configure Route 53 or ACM.
- Deleted resources after validation and documentation.

---

# Cleanup

Delete resources in the following order:

1. Auto Scaling Groups
2. Load Balancers
3. Target Groups
4. Launch Templates
5. EC2 Instances
6. AMIs (if created)
7. Snapshots
8. Amazon RDS
9. S3 Bucket
10. NAT Gateway
11. Elastic IP
12. Internet Gateway
13. Route Tables
14. Security Groups
15. VPC

---

# Conclusion

This project demonstrates the deployment of a production-inspired AWS 3-Tier Web Application Architecture using core AWS networking, compute, database, and security services.

The architecture includes:

- Custom VPC
- Public and Private Subnets
- Internet Gateway
- NAT Gateway
- External & Internal Application Load Balancers
- Web Tier Auto Scaling Group
- App Tier Auto Scaling Group
- Launch Templates
- Amazon RDS MySQL
- IAM Roles
- Nginx
- Node.js
- PM2
- React

The project follows AWS best practices for security, scalability, fault tolerance, and high availability while remaining suitable for a learning environment and portfolio demonstration.