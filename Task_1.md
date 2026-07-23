# AWS EC2 Launch Workflow

This document describes the standard workflow for launching an Amazon EC2 instance using AWS best practices. The process covers AMI selection, instance sizing, storage configuration, security, IAM permissions, and validation.

---

# EC2 Launch Workflow

```text
Start
   │
   ▼
Login to AWS Console
   │
   ▼
Select AWS Region
   │
   ▼
Launch EC2 Instance
   │
   ▼
Select AMI
   │
   ▼
Select Instance Type
   │
   ▼
Configure Networking
   │
   ▼
Configure IAM Role
   │
   ▼
Configure Storage (EBS)
   │
   ▼
Configure Security Group
   │
   ▼
Review Configuration
   │
   ▼
Launch Instance
   │
   ▼
Validate Instance
   │
   ▼
End
```

---

# 1. Launch EC2 Instance

## Purpose

Create a new virtual machine in AWS.

## Steps

- Open **AWS Management Console**
- Navigate to **EC2**
- Click **Launch Instance**

---

# 2. Select AMI (Amazon Machine Image)

## Purpose

An AMI provides the operating system and pre-installed software required to launch an EC2 instance.

## Common AMIs

| AMI | Use Case |
|------|----------|
| Amazon Linux 2023 | AWS-native workloads |
| Ubuntu Server LTS | General Linux applications |
| Red Hat Enterprise Linux | Enterprise environments |
| Debian | Lightweight Linux workloads |
| SUSE Linux | SAP and enterprise workloads |
| Microsoft Windows Server | Windows applications |

## Selection Criteria

- Operating system compatibility
- Long-term support (LTS)
- Required software
- Architecture (x86_64 or ARM64)
- Marketplace licensing (if applicable)

### Best Practices

- Use the latest supported AMI.
- Prefer official AWS or vendor-provided AMIs.
- Keep custom golden AMIs updated and patched.

---

# 3. Configure Instance Type

## Purpose

Choose the CPU, memory, networking, and storage performance characteristics of the instance.

## Workflow

```text
Application Requirements
          │
          ▼
Estimate CPU
          │
          ▼
Estimate Memory
          │
          ▼
Estimate Storage Performance
          │
          ▼
Estimate Network Performance
          │
          ▼
Choose Instance Family
```

## Common Instance Families

| Family | Purpose |
|---------|----------|
| t3 / t4g | General purpose, burstable |
| m7i / m6i | Balanced workloads |
| c7i / c6i | Compute-intensive |
| r7i / r6i | Memory-intensive |
| i4i | High IOPS local storage |
| g5 | GPU workloads |
| p5 | Machine learning and AI |

### Selection Factors

- vCPU requirements
- Memory requirements
- Network bandwidth
- Storage throughput
- Cost

---

# 4. Configure Networking

## Purpose

Place the instance into the appropriate network.

## Components

- VPC
- Subnet
- Public IP assignment
- Route Tables
- Internet Gateway
- NAT Gateway (for private instances)

## Public Instance

```text
Internet
    │
Internet Gateway
    │
Public Subnet
    │
EC2 Instance
```

## Private Instance

```text
Internet
    │
Internet Gateway
    │
NAT Gateway
    │
Private Subnet
    │
EC2 Instance
```

### Best Practices

- Place production workloads in private subnets.
- Assign public IPs only when necessary.
- Use Elastic IPs only for persistent public endpoints.

---

# 5. Configure IAM Role

## Purpose

Grant temporary AWS permissions to the EC2 instance without using access keys.

## Workflow

```text
Create IAM Role
       │
       ▼
Attach Policies
       │
       ▼
Assign Role to EC2
       │
       ▼
EC2 Receives Temporary Credentials
```

## Common IAM Policies

| Policy | Purpose |
|---------|----------|
| AmazonSSMManagedInstanceCore | Systems Manager access |
| AmazonS3ReadOnlyAccess | Read S3 objects |
| AmazonS3FullAccess | Full S3 access (avoid unless required) |
| CloudWatchAgentServerPolicy | Send metrics and logs |
| AmazonEC2ReadOnlyAccess | Read EC2 resources |

### Best Practices

- Use IAM Roles instead of access keys.
- Follow the principle of least privilege.
- Review and audit permissions regularly.

---

# 6. Configure EBS Volumes

## Purpose

Attach persistent block storage to the EC2 instance.

## Workflow

```text
Select Volume Type
       │
       ▼
Choose Size
       │
       ▼
Configure IOPS (if applicable)
       │
       ▼
Enable Encryption
       │
       ▼
Attach Volume
```

## Common Volume Types

| Volume Type | Use Case |
|-------------|----------|
| gp3 | General-purpose SSD (recommended) |
| io2 | High-performance SSD |
| st1 | Throughput-optimized HDD |
| sc1 | Cold HDD |
| Instance Store | Temporary local storage |

## Configuration Options

- Size (GiB)
- IOPS
- Throughput
- Encryption (AWS KMS)
- Delete on termination
- Snapshot source (optional)

### Best Practices

- Use **gp3** for most workloads.
- Enable encryption using AWS KMS.
- Use snapshots for backups.
- Avoid oversized volumes.

---

# 7. Configure Security Group

## Purpose

Control inbound and outbound network traffic for the EC2 instance.

## Workflow

```text
Create Security Group
       │
       ▼
Add Inbound Rules
       │
       ▼
Add Outbound Rules
       │
       ▼
Attach to EC2
```

## Example Inbound Rules

| Protocol | Port | Source | Purpose |
|----------|------|--------|----------|
| SSH | 22 | Your IP | Linux administration |
| RDP | 3389 | Your IP | Windows administration |
| HTTP | 80 | 0.0.0.0/0 | Web traffic |
| HTTPS | 443 | 0.0.0.0/0 | Secure web traffic |

## Best Practices

- Allow SSH/RDP only from trusted IP addresses.
- Avoid opening unnecessary ports.
- Use Security Group references for internal communication.
- Periodically review and remove unused rules.

---

# 8. Review Configuration

Before launching, verify:

- AMI
- Instance type
- VPC and subnet
- IAM role
- Security group
- EBS volume configuration
- Key pair (if SSH access is required)
- Monitoring settings
- Tags

---

# 9. Launch Instance

## Steps

- Click **Launch Instance**.
- Select or create a key pair (if applicable).
- Confirm the launch.

AWS provisions the instance and transitions it through:

```text
Pending
   │
   ▼
Running
```

---

# 10. Validate Instance

After the instance reaches the **Running** state:

## Connectivity

- SSH (Linux)
- RDP (Windows)
- Session Manager (recommended)

## Validation Checklist

- Instance status checks passed
- IAM role attached
- Security group rules verified
- EBS volume attached
- CloudWatch monitoring enabled
- Systems Manager managed (if applicable)
- Application is accessible

---

# Best Practices Summary

- Use the latest supported AMI.
- Select the smallest instance type that meets performance needs.
- Prefer **gp3** EBS volumes.
- Encrypt all EBS volumes with AWS KMS.
- Use IAM Roles instead of long-term access keys.
- Restrict Security Group rules to the minimum required.
- Place production instances in private subnets.
- Enable CloudWatch monitoring and Systems Manager.
- Apply consistent resource tags (e.g., `Name`, `Environment`, `Owner`, `CostCenter`).
- Regularly patch and update instances or use updated golden AMIs.

---

# Example Architecture

```text
                    Internet
                        │
                        ▼
                Internet Gateway
                        │
        ┌───────────────┴───────────────┐
        │                               │
        ▼                               ▼
   Public Subnet                  Private Subnet
        │                               │
        ▼                               ▼
   Bastion Host                  Application EC2
        │                               │
        └──────────────┬────────────────┘
                       ▼
                Amazon EBS (Encrypted)
                       │
                       ▼
                   IAM Role
                       │
                       ▼
          CloudWatch / Systems Manager
```
