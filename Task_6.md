
# AWS Lab: Deploy Multi-AZ EC2 Instances with Application Load Balancer (ALB)

## Objective

Deploy two EC2 instances in different Availability Zones, configure an
Application Load Balancer (ALB), enable health checks, and verify high
availability.

------------------------------------------------------------------------

# Architecture

``` text
                         Internet
                             │
                             ▼
                  Application Load Balancer
                       (HTTP : 80)
                             │
          ┌──────────────────┴──────────────────┐
          │                                     │
          ▼                                     ▼
   Public Subnet (AZ-A)                 Public Subnet (AZ-B)
      EC2-1 (Apache)                      EC2-2 (Apache)
      Blue Version                        Green Version
```

------------------------------------------------------------------------

# Prerequisites

-   AWS Account
-   Existing VPC
-   Internet Gateway attached
-   Public Route Table
-   Two Public Subnets in different AZs
-   Key Pair
-   IAM permissions to create EC2 and ALB

------------------------------------------------------------------------

# Step 1 - Create Security Groups

## ALB-SG

Inbound

  Type   Port   Source
  ------ ------ -----------
  HTTP   80     0.0.0.0/0

Outbound

-   All Traffic

------------------------------------------------------------------------

## EC2-SG

Inbound

  Type   Port   Source
  ------ ------ -----------------------
  SSH    22     My IP
  HTTP   80     0.0.0.0/0 (Temporary)

Outbound

-   All Traffic

> After testing, replace HTTP source with **ALB-SG**.

------------------------------------------------------------------------

# Step 2 - Launch EC2 Instance 1

-   Name: Web-Server-1
-   AMI: Amazon Linux 2023
-   Instance Type: t3.micro
-   VPC: Your VPC
-   Subnet: Public Subnet (AZ-A)
-   Auto Assign Public IP: Enable
-   Security Group: EC2-SG
-   Key Pair: Select existing key pair

Paste the following into **User Data**.

``` bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl enable httpd
systemctl start httpd

cat <<EOF > /var/www/html/index.html
<html>
<body style="background-color:#1E90FF;color:white;text-align:center;font-family:Arial;">
<h1>🟦 VERSION 2 (BLUE)</h1>
<h2>Server 1 - AZ-A</h2>
<h3>Application Load Balancer Demo</h3>
</body>
</html>
EOF
```

Launch the instance.

------------------------------------------------------------------------

# Step 3 - Launch EC2 Instance 2

-   Name: Web-Server-2
-   AMI: Amazon Linux 2023
-   Instance Type: t3.micro
-   VPC: Same VPC
-   Subnet: Public Subnet (AZ-B)
-   Auto Assign Public IP: Enable
-   Security Group: EC2-SG
-   Key Pair: Same key pair

User Data

``` bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl enable httpd
systemctl start httpd

cat <<EOF > /var/www/html/index.html
<html>
<body style="background-color:green;color:white;text-align:center;font-family:Arial;">
<h1>🟩 VERSION 2 (GREEN)</h1>
<h2>Server 2 - AZ-B</h2>
<h3>Application Load Balancer Demo</h3>
</body>
</html>
EOF
```

Launch the instance.

------------------------------------------------------------------------

# Step 4 - Verify Both Servers

Open:

    http://<EC2-1-Public-IP>

Expected:

    VERSION 2 (BLUE)
    Server 1 - AZ-A

Open:

    http://<EC2-2-Public-IP>

Expected:

    VERSION 2 (GREEN)
    Server 2 - AZ-B

------------------------------------------------------------------------

# Step 5 - Create Target Group

Navigate to:

EC2 → Target Groups → Create Target Group

Configuration

-   Target Type: Instances
-   Name: web-target-group
-   Protocol: HTTP
-   Port: 80
-   VPC: Select your VPC
-   Health Check Protocol: HTTP
-   Health Check Path: /

Click **Next**.

Register:

-   Web-Server-1
-   Web-Server-2

Click **Create Target Group**.

------------------------------------------------------------------------

# Step 6 - Create Application Load Balancer

Navigate to:

EC2 → Load Balancers → Create Load Balancer

Choose:

Application Load Balancer

Configuration

-   Name: my-alb
-   Scheme: Internet-facing
-   IP Address Type: IPv4

Network Mapping

-   Select your VPC
-   Select Public Subnet (AZ-A)
-   Select Public Subnet (AZ-B)

Security Group

-   ALB-SG

Listener

-   HTTP
-   Port 80

Default Action

-   Forward to web-target-group

Click **Create Load Balancer**.

Wait until the status becomes **Active**.

------------------------------------------------------------------------

# Step 7 - Verify Target Health

Open:

EC2 → Target Groups → web-target-group → Targets

Expected

    Web-Server-1    Healthy
    Web-Server-2    Healthy

------------------------------------------------------------------------

# Step 8 - Test Load Balancing

Copy the ALB DNS Name.

Open it in a browser.

Refresh multiple times.

Expected:

    VERSION 2 (BLUE)
<img width="3276" height="1911" alt="image" src="https://github.com/user-attachments/assets/2629761e-f71d-4a49-9ec0-82db054ff503" />

Refresh

    VERSION 1 (GREEN)
<img width="3335" height="1782" alt="image" src="https://github.com/user-attachments/assets/3b2a8b51-160f-42f5-9bcb-4566b1c90ec1" />

Refresh again

    VERSION 2 (BLUE)

------------------------------------------------------------------------

# Step 9 - Test Health Checks

SSH into EC2-1.

``` bash
sudo systemctl stop httpd
```

Wait approximately one minute.

Open:

Target Groups → Targets

Expected:

    Web-Server-1    Unhealthy
    Web-Server-2    Healthy

Refresh the ALB DNS.

Only the Green page should appear.

Restart Apache.

``` bash
sudo systemctl start httpd
```

After a short time, both targets become Healthy again.

------------------------------------------------------------------------

# Step 10 - Secure the Environment

Edit EC2-SG.

Remove

  Type   Port   Source
  ------ ------ -----------
  HTTP   80     0.0.0.0/0

Add

  Type   Port   Source
  ------ ------ --------
  HTTP   80     ALB-SG

Now HTTP traffic reaches EC2 only through the ALB.

------------------------------------------------------------------------

# Troubleshooting

  -----------------------------------------------------------------------
  Problem                             Solution
  ----------------------------------- -----------------------------------
  Target Unhealthy                    Ensure httpd is running
                                      (`sudo systemctl status httpd`)

  Health Check Failed                 Verify path `/` returns HTTP 200

  ALB returns 503                     Confirm at least one healthy target

  Cannot access EC2                   Check Security Group, Route Table
                                      and Internet Gateway

  ALB not reachable                   Verify ALB-SG allows HTTP 80
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Cleanup

Delete resources in this order:

1.  Application Load Balancer
2.  Target Group
3.  EC2 Instances
4.  Security Groups (if no longer required)

------------------------------------------------------------------------

# Expected Outcome

-   Two EC2 instances running in different Availability Zones.
-   Application Load Balancer distributing traffic.
-   Health checks automatically removing unhealthy instances.
-   High availability across multiple AZs.
-   Secure production-ready traffic flow through the ALB.
