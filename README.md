# 🚀 **AWS VPC Project (Beginner → Advanced)**

**Project Title:** *Secure Multi-Tier VPC Infrastructure on AWS*
**Goal:** Build a fully-functional, secure, highly-available VPC with public/private subnets, NAT Gateway, routing, security groups, and optional EC2/Bastion deployment.

---

# ✅ **1. Project Overview**

You will create a **custom AWS Virtual Private Cloud (VPC)** using best practices:

### **Architecture Includes**

* Custom VPC (CIDR: `10.0.0.0/16`)
* 2 Public Subnets (AZ-A, AZ-B)
* 2 Private Subnets (AZ-A, AZ-B)
* Internet Gateway (IGW)
* NAT Gateway (for private internet access)
* Public & Private Route Tables
* Security Groups:

  * Bastion Host SG
  * Private EC2 SG
* Optional: EC2 Bastion + Private App Server

### **High-Level Diagram**

```
                 +------------------------------+
                 |          AWS VPC             |
                 |         10.0.0.0/16          |
+------------+   |   +-------------+   +-------------+
| Internet   |---|---| Public SubA |---| Public SubB |
+------------+   |   | 10.0.1.0/24 |   | 10.0.2.0/24 |
                 |   +------+------+   +------+------+
                 |          | NAT GW           |
                 |   +-------------+   +-------------+
                 |   | Private SubA|   | Private SubB|
                 |   |10.0.3.0/24  |   |10.0.4.0/24  |
                 +---+-------------+---+-------------+
```

---

# 🎯 **2. Project Requirements**

### **Prerequisites**

* AWS Account
* IAM user with **EC2, VPC, IAM, Subnet, Route53** permissions
* AWS CLI installed
* SSH Key

---

# 🔧 **3. Step-by-Step Implementation**

---

## **Step 1: Create VPC**

```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=demo-vpc}]'
```

---

## **Step 2: Create Subnets**

### **Public Subnet A**

```bash
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --availability-zone ap-south-1a \
  --cidr-block 10.0.1.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-a}]'
```

### **Public Subnet B**

```bash
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --availability-zone ap-south-1b \
  --cidr-block 10.0.2.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-b}]'
```

### **Private Subnet A**

```bash
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --availability-zone ap-south-1a \
  --cidr-block 10.0.3.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-a}]'
```

### **Private Subnet B**

```bash
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --availability-zone ap-south-1b \
  --cidr-block 10.0.4.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-b}]'
```

---

## **Step 3: Create Internet Gateway**

```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=demo-igw}]'
```

Attach it to VPC:

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id <igw-id> \
  --vpc-id <vpc-id>
```

---

## **Step 4: Create NAT Gateway**

### **Allocate Elastic IP**

```bash
aws ec2 allocate-address \
  --domain vpc
```

### **Create NAT Gateway**

```bash
aws ec2 create-nat-gateway \
  --subnet-id <public-subnet-a> \
  --allocation-id <allocation-id> \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=demo-natgw}]'
```

---

## **Step 5: Create Route Tables**

### **Public Route Table**

```bash
aws ec2 create-route-table \
  --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'
```

Add route to Internet:

```bash
aws ec2 create-route \
  --route-table-id <public-rt-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <igw-id>
```

Associate with Public Subnets:

```bash
aws ec2 associate-route-table \
  --subnet-id <public-subnet-a> \
  --route-table-id <public-rt-id>

aws ec2 associate-route-table \
  --subnet-id <public-subnet-b> \
  --route-table-id <public-rt-id>
```

### **Private Route Table**

```bash
aws ec2 create-route-table \
  --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]'
```

Add route to NAT Gateway:

```bash
aws ec2 create-route \
  --route-table-id <private-rt-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id <nat-id>
```

Associate with Private Subnets:

```bash
aws ec2 associate-route-table \
  --subnet-id <private-subnet-a> \
  --route-table-id <private-rt-id>

aws ec2 associate-route-table \
  --subnet-id <private-subnet-b> \
  --route-table-id <private-rt-id>
```

---

# 🔐 **6. Create Security Groups**

### **Bastion Host SG**

```bash
aws ec2 create-security-group \
  --group-name bastion-sg \
  --description "Allow SSH" \
  --vpc-id <vpc-id>
```

Allow SSH:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <bastion-sg> \
  --protocol tcp --port 22 --cidr 0.0.0.0/0
```

---

### **Private EC2 SG (Allow only Bastion)**

```bash
aws ec2 create-security-group \
  --group-name private-ec2-sg \
  --description "Private EC2 access from Bastion" \
  --vpc-id <vpc-id>
```

Allow SSH inside VPC:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <private-ec2-sg> \
  --protocol tcp --port 22 \
  --source-group <bastion-sg>
```

---

# 💻 **7. Optional: Launch EC2 Instances**

### **Bastion Host in Public Subnet**

```bash
aws ec2 run-instances \
  --image-id ami-0e670eb768a5eb91d \
  --count 1 \
  --instance-type t2.micro \
  --key-name mykey \
  --security-group-ids <bastion-sg> \
  --subnet-id <public-subnet-a> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=bastion-host}]'
```

### **App Server in Private Subnet**

```bash
aws ec2 run-instances \
  --image-id ami-0e670eb768a5eb91d \
  --count 1 \
  --instance-type t2.micro \
  --key-name mykey \
  --security-group-ids <private-ec2-sg> \
  --subnet-id <private-subnet-a> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=app-server}]'
```

---

# 🧪 **8. Testing & Validation**

### **Test Internet from Public Subnet**

```
curl google.com
```

### **Test from Private Instance**

```bash
curl google.com 
# Should work via NAT Gateway
```

### **Verify Subnet Routing**

```
aws ec2 describe-route-tables
```

---

# 📘 **9. README for GitHub**

I can generate a complete **README.md** for your GitHub project — just say **“create README”**.

---

# 🧱 **10. Terraform Version of This Project**

If you want **full Terraform automation**, say:
👉 **“Create Terraform code for this AWS VPC project”**

---

# 🎁 **11. Zip Folder Structure**

* scripts
* diagrams
* README
* terraform
* CloudFormation
