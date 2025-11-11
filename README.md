#   Terraform Project 

**Author:** Hany Mohamed Shalaby Elshafey

**Track:** System Administration

**Project:** AWS Reverse Proxy Infrastructure

#  AWS Reverse Proxy Infrastructure — Terraform Deployment

This project provisions a robust AWS environment using Terraform modules, featuring Nginx reverse proxies, Flask backend services, and Application Load Balancers (ALB).
All infrastructure setup, deployment, and configuration is fully automated.

---

## Architecture Overview

```
Internet → Public ALB → Nginx Reverse Proxies (Public Subnets)
                             ↓
                     Internal ALB → Flask Backends (Private Subnets)
```

### Components

- **VPC** — Custom VPC (CIDR: 10.0.0.0/16) spanning two Availability Zones  
- **Public Subnets** — Host Nginx proxy instances  
- **Private Subnets** — Run backend Flask applications  
- **NAT Gateway** — Allows private instances to access the Internet  
- **Public ALB** — Entry point for external users  
- **Internal ALB** — Balances traffic between proxies and backends  
- **EC2 Instances** — Two proxies and two backends, configured via provisioners  

---

## Prerequisites

Before deploying, ensure you have:

1. An **AWS Account** with sufficient IAM permissions  
2. **Terraform v1.0+** installed  
3. **AWS CLI** configured via `aws configure`  
4. A valid **EC2 SSH key pair**  
5. An **S3 bucket** for Terraform remote state  
6. A **DynamoDB table** for state locking  

---

## Quick Deployment Guide

### Step one : Setup Remote Backend

Create S3 and DynamoDB resources for Terraform state management:

```bash
aws s3api create-bucket   --bucket reverse-proxy-terraform-state-mohamed   --region eu-west-1   --create-bucket-configuration LocationConstraint=eu-central-1

aws s3api put-bucket-versioning   --bucket reverse-proxy-terraform-state-mohamed   --versioning-configuration Status=Enabled

aws dynamodb create-table   --table-name terraform-state-lock   --attribute-definitions AttributeName=LockID,AttributeType=S   --key-schema AttributeName=LockID,KeyType=HASH   --billing-mode PAY_PER_REQUEST   --region eu-west-1
```

---

### Step two : Configure Variables

Open and update `terraform.tfvars` with your own values:

```bash
nano terraform.tfvars
```

Required updates:
- `key_name` — your EC2 key pair name  
- `private_key_path` — path to your `.pem` file  
- Update S3 bucket name in `backend.tf`  

---

### Step three : Initialize Workspace

```bash
terraform init
terraform workspace new dev
terraform workspace select dev
```

---

### Step four : Deploy the Infrastructure

```bash
terraform plan
terraform apply
```

---

## Project Structure

```
terraform-aws-reverse-proxy/
├── main.tf                 # Root configuration
├── variables.tf            # Global variables
├── outputs.tf              # Global outputs
├── providers.tf            # AWS provider setup
├── backend.tf              # Remote backend config
├── terraform.tfvars        # Environment variables
│
├── modules/
│   ├── network/            # VPC, Subnets, IGW, NAT
│   ├── security/           # Security Groups
│   ├── compute/            # EC2 + Provisioners
│   └── loadbalancing/      # ALBs & Target Groups
│
├── provisioners/
│   └── app/
│       ├── app.py          # Flask backend app
│       └── requirements.txt
│
└── all-ips.txt             # Generated instance IP list
```

---

## Provisioners

### Proxy Instances (Public)
- Installs and configures **Nginx**
- Sets up reverse proxy to internal ALB  
- Runs remotely via SSH  

### Backend Instances (Private)
- Installs **Python3** and **Flask**
- Starts the Flask app on port 80  
- Connects through proxy (bastion host)  

### File Provisioner
Copies the application files (`app.py`, `requirements.txt`) to backend instances.

### Local-Exec
Generates a list of IPs in `all-ips.txt`:
```
proxy-ip1 54.123.45.67
proxy-ip2 54.123.45.68
backend-ip1 10.0.1.10
backend-ip2 10.0.3.20
```

---

## Testing & Verification

### Check Outputs
```bash
terraform output public_alb_url
```

### Test Access
```bash
curl $(terraform output -raw public_alb_url)
```

Expected response:
```json
{
  "status": "healthy",
  "message": "Backend server is running!",
  "hostname": "ip-10-0-1-10",
  "ip": "10.0.1.10",
  "environment": "dev"
}
```

### View IP List
```bash
cat all-ips.txt
```

### SSH Access
```bash
ssh -i ~/your-key.pem ec2-user@<proxy-public-ip>
ssh -i ~/your-key.pem -J ec2-user@<proxy-public-ip> ec2-user@<backend-private-ip>
```

---

## Security Best Practices

1. Restrict SSH access to trusted IPs only  
2. Use **IAM roles** instead of embedding credentials  
3. Enable **MFA** on your AWS account  
4. Rotate SSH keys regularly  
5. Enable **VPC Flow Logs** for auditing  
6. Use **HTTPS** in production environments  

---

## Cleanup

```bash
terraform destroy -auto-approve
terraform workspace select default
terraform workspace delete dev
```

---

## Estimated Monthly Cost (us-east-1)

| Resource | Type | Approx. Cost |
|-----------|------|--------------|
| EC2 (4 × t3.micro) | Compute | ~$30 |
| NAT Gateway | Networking | ~$32 |
| ALB (2 ×) | Load Balancers | ~$32 |
| **Total (approx.)** |  | **~$95/month** |

---

## Technical Notes

- Uses `aws_ami` data source for latest **Amazon Linux 2**
- Supports **Terraform workspaces** (`dev`, etc.)
- State managed remotely in **S3 + DynamoDB**
- Proxies double as **bastion hosts**
- ALBs perform HTTP health checks on `/`

---

## Module Summary

| Module | Description |
|---------|-------------|
| **Network** | Builds VPC, subnets, IGW, NAT, and routing |
| **Security** | Creates all necessary security groups |
| **Compute** | Launches EC2s and runs provisioners |
| **Load Balancing** | Creates and configures ALBs & target groups |

---

 
