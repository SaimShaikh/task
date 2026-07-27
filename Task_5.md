
# AWS Systems Manager (SSM) Agent Installation & Session Manager — Login to Private EC2 Instance (No SSH, No Bastion, No Public IP)

---

---

## 1. Concept Overview

AWS Systems Manager **Session Manager** lets you get an interactive shell (or PowerShell) session on an EC2 instance **without**:

- Opening inbound port 22 (SSH) or 3389 (RDP)
- Assigning a public IP address
- Managing SSH key pairs
- Using a bastion/jump host
- Any inbound security group rule at all

Instead, the **SSM Agent** running on the instance makes an **outbound** connection to the AWS Systems Manager service. When you start a session from the console (or CLI), Systems Manager brokers a secure, encrypted, bidirectional channel between your terminal and the instance's shell — entirely over that outbound connection.

**Core components involved:**

| Component | Role |
|---|---|
| **SSM Agent** | Software running on the EC2 instance that talks to the SSM service |
| **IAM Role (Instance Profile)** | Grants the EC2 instance permission to talk to SSM |
| **AmazonSSMManagedInstanceCore** | AWS managed policy that provides the minimum permissions for SSM Agent to function |
| **VPC Interface Endpoints** | Required when the instance has no internet route (no NAT Gateway/IGW) — lets the agent reach SSM privately via AWS PrivateLink |
| **Session Manager** | The Systems Manager feature that brokers the actual shell session |
| **CloudTrail (optional)** | Logs every session start/end — who connected, when |

---

## 2. Why This Matters (The Problem With SSH + Bastion Hosts)

**Traditional private instance access:**

```
Your Laptop → Bastion Host (public IP, SSH open) → Private Instance (SSH)
```

Problems with this model:

- Bastion host is a **permanent attack surface** — it has a public IP and an open SSH port 24/7.
- You must **manage SSH key pairs**, rotate them, and worry about leaked keys.
- You need **security group chaining** (bastion SG → private instance SG).
- No **native session logging** — you'd need to bolt on your own auditing (script session recording, etc.).
- **Compliance headaches** — proving "who accessed what, when" is manual work.

**With SSM Session Manager:**

```
Your Laptop → AWS Systems Manager Service → Private Instance (SSM Agent, outbound only)
```

- **Zero inbound ports.** No SSH, no RDP, no bastion.
- **No SSH keys** to manage or rotate.
- **IAM controls access** — the same IAM policies you already use for AWS.
- **Every session is logged** in CloudTrail; session content can be logged to S3/CloudWatch Logs.
- Works even if the instance is in a **fully private subnet with zero internet access**, as long as VPC Interface Endpoints exist.

---

## 3. Architecture Diagram

### Scenario A: Private subnet WITH NAT Gateway (agent reaches SSM via NAT → IGW)

```
                                   ┌─────────────────────────────────────────┐
                                   │                  AWS Cloud               │
                                   │                                           │
  ┌────────────┐                  │   ┌───────────────────────────────────┐   │
  │  IAM User   │  Console/CLI     │   │           VPC (10.0.0.0/16)        │   │
  │ (You)       │─────────────────┼──▶│                                     │   │
  └────────────┘                  │   │  ┌─────────────────────────────┐    │   │
                                   │   │  │      Public Subnet          │    │   │
                                   │   │  │   ┌───────────────────┐     │    │   │
                                   │   │  │   │   NAT Gateway      │     │    │   │
                                   │   │  │   └─────────┬─────────┘     │    │   │
                                   │   │  └─────────────┼───────────────┘    │   │
                                   │   │                │                    │   │
                                   │   │  ┌─────────────┼───────────────┐    │   │
                                   │   │  │   Private Subnet            │    │   │
                                   │   │  │   ┌─────────▼─────────┐     │    │   │
                                   │   │  │   │  EC2 Instance      │     │    │   │
                                   │   │  │   │  - No Public IP    │     │    │   │
                                   │   │  │   │  - SSM Agent (out) │     │    │   │
                                   │   │  │   │  - IAM Role attach │     │    │   │
                                   │   │  │   └────────────────────┘     │    │   │
                                   │   │  └───────────────────────────────┘    │   │
                                   │   │                                     │   │
                                   │   └───────────────┬─────────────────────┘   │
                                   │                   │ (via Internet Gateway)  │
                                   │           ┌────────▼────────┐               │
                                   │           │  AWS Systems     │               │
                                   │           │  Manager Service │               │
                                   │           └──────────────────┘               │
                                   └─────────────────────────────────────────┘
```

### Scenario B: FULLY PRIVATE subnet — NO NAT, NO IGW (uses VPC Interface Endpoints) — **this is the scenario this lab builds**

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                  AWS Cloud                                │
│                                                                            │
│   ┌────────────┐         ┌────────────────────────────────────────────┐  │
│   │  IAM User   │  HTTPS  │              VPC (10.0.0.0/16)              │  │
│   │  (Console)  │────────▶│                                              │  │
│   └────────────┘         │   ┌──────────────────────────────────────┐   │  │
│                           │   │        Private Subnet (10.0.1.0/24)  │   │  │
│                           │   │                                       │   │  │
│                           │   │   ┌─────────────────────────────┐    │   │  │
│                           │   │   │   EC2 Instance               │    │   │  │
│                           │   │   │   - No Public IP              │    │   │  │
│                           │   │   │   - No NAT/IGW route          │    │   │  │
│                           │   │   │   - SSM Agent (pre-installed) │    │   │  │
│                           │   │   │   - IAM Instance Profile      │    │   │  │
│                           │   │   └──────────────┬────────────────┘    │   │  │
│                           │   │                  │ ENI (private IP)    │   │  │
│                           │   │                  ▼                     │   │  │
│                           │   │   ┌─────────────────────────────┐    │   │  │
│                           │   │   │   VPC Interface Endpoints     │    │   │  │
│                           │   │   │   (PrivateLink, in same VPC)  │    │   │  │
│                           │   │   │                                │    │   │  │
│                           │   │   │  • com.amazonaws.<region>.ssm  │    │   │  │
│                           │   │   │  • com.amazonaws.<region>.ec2messages │  │
│                           │   │   │  • com.amazonaws.<region>.ssmmessages │  │
│                           │   │   └──────────────┬────────────────┘    │   │  │
│                           │   └──────────────────┼─────────────────────┘   │  │
│                           │                      │                          │  │
│                           │                      ▼                          │  │
│                           │            AWS Systems Manager Service          │  │
│                           │            (reached via PrivateLink,            │  │
│                           │             traffic never leaves AWS network)   │  │
│                           └──────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Prerequisites

- AWS account with console access
- IAM permissions to create: IAM roles, EC2 instances, VPCs/Subnets, Security Groups, VPC Endpoints
- A VPC with at least one **private subnet** (this lab builds one from scratch — you can also reuse an existing VPC)
- Region used in this lab: **ap-south-1 (Mumbai)** — substitute your own region, but keep it consistent everywhere
- No SSH key pair needed
- No AWS CLI required for the lab steps (a CLI reference section is included separately at the end for those who want it)

---

## ⚠️ READ THIS BEFORE YOU START — Which Path Are You Doing?

This lab is written for **Scenario B: a fully private subnet with NO NAT Gateway and NO Internet Gateway**. That is the harder, more realistic enterprise pattern, and it is the whole reason **Step 3 (VPC Interface Endpoints) is mandatory, not optional, in this guide.**

Confirm which path applies to you **before you launch the EC2 instance**, because it changes whether Step 3 is required:

| Your Subnet Has... | Path | Do You Need Step 3 (VPC Endpoints)? |
|---|---|---|
| A route to an Internet Gateway directly (public subnet) | Scenario A | ❌ No — skip Step 3 entirely, go straight to Step 4 |
| A route to a NAT Gateway (private subnet, but NAT exists) | Scenario A | ❌ No — skip Step 3 entirely, go straight to Step 4 |
| **No NAT Gateway and no IGW route at all** (fully isolated) | **Scenario B (this guide)** | ✅ **Yes — mandatory.** Without it, the SSM Agent has no path to the SSM service, the instance will never show "Online" in Fleet Manager, and Session Manager will never connect. |

**This guide builds Scenario B from scratch**, including creating a subnet with zero NAT/IGW route on purpose, specifically so you get hands-on practice with VPC Interface Endpoints — this is the part most tutorials skip and the part interviewers actually ask about.

**Sequence matters — this is why the steps are ordered the way they are:** Step 1 (IAM Role) → Step 2 (Launch Instance) → Step 3 (VPC Endpoints) → Step 4 (Verify Agent/Fleet Manager) → Step 5 (Connect). Endpoints are created *before* you ever check Fleet Manager, so you never see a confusing "0 managed nodes" screen for something that isn't actually broken yet.

If you'd rather do the simpler version first, use a subnet that already has a NAT Gateway, skip Step 3 entirely, and go straight from Step 2 to Step 4.

---

## 5. Lab Setup — Building the Environment

We will build this from scratch so the "fully private, no internet route" scenario is proven end-to-end:

1. A VPC
2. A private subnet (no route to IGW/NAT)
3. A Security Group for the EC2 instance (no inbound rules needed at all)
4. A Security Group for the VPC endpoints (allows HTTPS inbound from the instance SG)
5. An IAM Role for SSM
6. An EC2 instance in the private subnet
7. Three VPC Interface Endpoints (ssm, ssmmessages, ec2messages)
8. A Session Manager connection

If you already have a VPC/private subnet you want to reuse, skip to **Step 1** and just make sure your chosen subnet has no route to a NAT Gateway or Internet Gateway (to genuinely test the private scenario), or skip endpoint creation entirely if your subnet already routes to a NAT Gateway.

### 5.1 Create the VPC

1. Go to **VPC console** → **Your VPCs** → **Create VPC**
2. Choose **VPC only**
3. **Name tag:** `ssm-lab-vpc`
4. **IPv4 CIDR block:** `10.0.0.0/16`
5. Leave IPv6 as **No IPv6 CIDR block**
6. Tenancy: **Default**
7. Click **Create VPC**

### 5.2 Create the Private Subnet

1. In VPC console → **Subnets** → **Create subnet**
2. **VPC ID:** select `ssm-lab-vpc`
3. **Subnet name:** `ssm-private-subnet`
4. **Availability Zone:** pick any, e.g. `ap-south-1a`
5. **IPv4 CIDR block:** `10.0.1.0/24`
6. Click **Create subnet**
7. Confirm this subnet's **route table** only has the default `local` route (no `0.0.0.0/0` entry pointing to an IGW or NAT). This is what makes it "private."

---

## 6. Step 1: Create IAM Role for SSM

The EC2 instance needs an IAM role so the SSM Agent on it can authenticate to the Systems Manager service.

1. Go to **IAM console** → **Roles** → **Create role**
2. **Trusted entity type:** AWS service
3. **Use case:** select **EC2** → click **Next**
4. In the permissions policy search box, type: `AmazonSSMManagedInstanceCore`
5. Check the box next to **AmazonSSMManagedInstanceCore**
6. Click **Next**
7. **Role name:** `EC2-SSM-Role`
8. Review and click **Create role**


These are exactly the API calls the SSM Agent makes internally — nothing more.

---

## 7. Step 2: Launch the Private EC2 Instance

1. Go to **EC2 console** → **Instances** → **Launch instances**
2. **Name:** `ssm-private-instance`
3. **AMI:** Amazon Linux 2023 (SSM Agent comes **pre-installed** on this AMI — ideal for this lab)
4. **Instance type:** `t2.micro` or `t3.micro` (Free Tier eligible)
5. **Key pair:** select **Proceed without a key pair** (not needed — this is the whole point of SSM!)
6. **Network settings** → click **Edit**:
   - **VPC:** `ssm-lab-vpc`
   - **Subnet:** `ssm-private-subnet`
   - **Auto-assign public IP:** **Disable**
   - **Security group:** Create a new one:
     - **Name:** `ssm-instance-sg`
     - **Inbound rules:** **none** (leave empty — delete the default SSH rule if present)
     - **Outbound rules:** leave default (Allow all outbound) — the agent needs outbound HTTPS (443)
7. Expand **Advanced details**:
   - **IAM instance profile:** select `EC2-SSM-Role`
8. Leave storage as default
9. Click **Launch instance**
10. Wait until **Instance state = Running** and **Status checks = 2/2 checks passed**


<img width="2124" height="1682" alt="image" src="https://github.com/user-attachments/assets/d5069d4b-bb88-4774-bc1e-dcb488460806" />
<img width="2124" height="1682" alt="image" src="https://github.com/user-attachments/assets/7866fc2f-eb05-4c33-8719-3fa30873b59f" />
<img width="2034" height="956" alt="image" src="https://github.com/user-attachments/assets/92870a67-cd6f-4dd0-af2f-793a102e25d1" />

---

## 8. Step 3: Set Up VPC Interface Endpoints (Required for Private Subnet — No NAT/IGW)

Do this **before** checking Fleet Manager. Since `ssm-private-subnet` has **no route to the internet** (no NAT Gateway, no IGW), the SSM Agent cannot reach the public SSM service endpoints — and it never will, no matter how long you wait, until this step is done. We fix this using **AWS PrivateLink / VPC Interface Endpoints**, which place SSM's endpoints directly inside your VPC.

**Three endpoints are required together** (all three, not just one):

| Endpoint | Purpose |
|---|---|
| `com.amazonaws.<region>.ssm` | Core Systems Manager API calls |
| `com.amazonaws.<region>.ssmmessages` | Session Manager's actual data channel (the shell traffic) |
| `com.amazonaws.<region>.ec2messages` | Agent-to-service messaging used by SSM |




### 8.1 Create the Endpoint Security Group

1. Go to **EC2 console** → **Security Groups** → **Create security group**
2. **Name:** `ssm-endpoint-sg`
3. **VPC:** `ssm-lab-vpc`
4. **Inbound rules:**
   - Type: **HTTPS**, Port: **443**, Source: `ssm-instance-sg` (select the security group, not an IP range)
5. **Outbound rules:** leave default (allow all)
6. Click **Create security group**

### 8.2 Create the Three Interface Endpoints

Repeat this three times, once per service name.

1. Go to **VPC console** → **Endpoints** → **Create endpoint**
2. **Name tag:** `vpce-ssm` (then `vpce-ssmmessages`, then `vpce-ec2messages` on the next two runs)
3. **Service category:** AWS services
4. **Service name:** search and select:
   - Run 1: `com.amazonaws.<your-region>.ssm`
   - Run 2: `com.amazonaws.<your-region>.ssmmessages`
   - Run 3: `com.amazonaws.<your-region>.ec2messages`
5. **VPC:** `ssm-lab-vpc`
6. **Subnets:** select the AZ containing `ssm-private-subnet` (check the box for that AZ)
7. **Security groups:** uncheck default, check `ssm-endpoint-sg`
8. **Policy:** Full access (default)
9. Click **Create endpoint**
10. Repeat for the remaining two service names


<img width="3302" height="624" alt="image" src="https://github.com/user-attachments/assets/9d781e13-bdc7-4117-9527-a0d75176d44d" />

> Wait until all three endpoints show **Status: Available** (takes 1–3 minutes each) before proceeding to Step 4.

### 8.3 Confirm DNS Resolution Is Enabled

1. Select `ssm-lab-vpc` in the VPC console
2. **Actions** → **Edit VPC settings**
3. Ensure **Enable DNS resolution** and **Enable DNS hostnames** are both checked
4. Save if you changed anything

This step matters because the SSM Agent resolves the SSM service hostname, and that hostname must resolve to the **private** endpoint IP (not a public IP) inside the VPC — which only works correctly when DNS resolution is enabled on the VPC.

> **If you're on Scenario A** (your subnet already has a NAT Gateway or IGW route), skip this entire Step 3 — go straight to Step 4.

---

## 9. Step 4: Verify SSM Agent Is Installed & Running

Amazon Linux 2023, Amazon Linux 2, and Ubuntu 20.04+ AMIs from AWS come with the SSM Agent **pre-installed**. You do not need to install it manually on these AMIs. Now that the endpoints exist (or you're on a subnet with NAT/IGW), the agent has a real path to reach SSM — this is the point where it's actually meaningful to check.

### 9.1 Check via Systems Manager Console (Fleet Manager)

1. Go to **Systems Manager console** → left sidebar → **Fleet Manager** (under Node Management)
2. Look for `ssm-private-instance` in the managed instances list
3. **Ping status** should show **Online** (allow 1–2 minutes after endpoints become Available)
<img width="3211" height="1961" alt="image" src="https://github.com/user-attachments/assets/686d4597-9558-45a9-9c4c-6e4d1480d093" />

<img width="3210" height="1185" alt="image" src="https://github.com/user-attachments/assets/2937d71f-61ca-424c-a4ec-21eeeeb31313" />


> If it's still not appearing after a few minutes, that's now a real issue, not an expected wait

### 9.2 (If Needed) Manually Verify/Install/Restart the Agent on the Instance Itself

You will only need this section if:
- You used a custom/older AMI where the agent isn't pre-installed, OR
- Fleet Manager still shows the instance as offline after the endpoints are Available, and you need to debug from inside

Since the instance has no SSH/public IP, you can only run these commands **after** you already have Session Manager access (chicken-and-egg only applies to brand-new custom AMIs — standard Amazon Linux/Ubuntu AMIs already have it pre-installed and running, so Step 5 below will just work once it's Online).

**Check agent status (Amazon Linux 2023 / Amazon Linux 2 / RHEL):**
```bash
sudo systemctl status amazon-ssm-agent
```

**Check agent status (Ubuntu):**
```bash
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
```

**Start the agent if stopped:**
```bash
sudo systemctl start amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
```

**Manually install the agent (Amazon Linux, if genuinely missing):**
```bash
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

**Manually install the agent (Ubuntu, if genuinely missing):**
```bash
sudo snap install amazon-ssm-agent --classic
sudo snap start amazon-ssm-agent
```

**Check the agent's version:**
```bash
sudo systemctl status amazon-ssm-agent | grep "Active"
amazon-ssm-agent -version
```

**View agent logs (for debugging registration issues):**
```bash
sudo tail -f /var/log/amazon/ssm/amazon-ssm-agent.log
```

---

## 10. Step 5: Connect to the Private Instance Using Session Manager

Now the actual login.

1. Go to **EC2 console** → **Instances**
2. Select the checkbox next to `ssm-private-instance`
3. Click **Connect** (top right)
4. In the **Connect to instance** page, select the **Session Manager** tab
5. You should see: *"Amazon EC2 Instance Connect and Session Manager"* with your instance listed and eligible
6. Click **Connect**
<img width="2" height="2" alt="image" src="https://github.com/user-attachments/assets/1795f5a5-9548-45cd-8403-57fed1336617" />
<img width="3312" height="979" alt="image" src="https://github.com/user-attachments/assets/55e6093e-b79a-44b6-8a82-1eccb996c12a" />

A new browser tab opens with a **black terminal window** — you are now inside the private EC2 instance's shell, with:

- No SSH client used
- No public IP on the instance
- No inbound security group rule
- No key pair

You'll land in a shell as the `ssm-user` (a user automatically created by the SSM Agent for session access).

---



## 11. Edge Cases & Failure Scenarios

| Scenario | What Happens | Root Cause | Fix |
|---|---|---|---|
| Instance not visible in Fleet Manager | "Managed instance" never appears | IAM role missing/wrong, or agent not running | Attach `AmazonSSMManagedInstanceCore` to instance role; verify agent status |
| Fleet Manager shows "Connection lost" | Was online, now offline | Agent crashed, instance stopped/started with role detached, or network path broken | Restart agent; check instance profile is still attached; check endpoint health |
| "Connect" button greyed out / Session Manager tab missing | Console won't let you start a session | No IAM role attached at all, or agent never registered | Attach IAM role; may require instance reboot for role attachment to take effect in some agent versions |
| Session starts but immediately terminates | Session opens then closes in 1–2 seconds | SSM document `SSM-SessionManagerRunShell` missing/misconfigured, or agent version too old | Update SSM Agent; check Session Manager preferences under Systems Manager → Session Manager → Preferences |
| Private subnet + no VPC endpoints created | Instance never registers as "Online" | No path to reach SSM service (no NAT/IGW and no PrivateLink) | Create all 3 required interface endpoints (ssm, ssmmessages, ec2messages) |
| Endpoints created but instance still offline | Same as above, persists | Endpoint security group doesn't allow inbound 443 from instance's SG, or endpoint deployed in wrong AZ/subnet | Fix endpoint SG inbound rule; ensure endpoint subnet matches instance's AZ |
| DNS resolution disabled on VPC | Endpoints exist but agent can't find them | VPC-level "Enable DNS resolution" or "Enable DNS hostnames" turned off | Enable both under VPC settings |
| Old/custom AMI without SSM Agent | Instance boots, but never appears in Fleet Manager | Agent was never installed at image build time | Bake agent into custom AMI at build time, or install via user-data script at launch |
| IAM role attached to instance AFTER launch | Instance still not appearing | Attaching a role after launch sometimes requires the agent to restart to pick up new credentials | Reboot instance, or (if you already have a session another way) restart the agent manually |
| Session Manager logging required for compliance | No record of what commands were run | Session logging wasn't enabled | Enable session logging to S3/CloudWatch Logs under Session Manager → Preferences |
| Multiple users need session access | Uncontrolled access across team | No IAM policy scoping who can start sessions on which instances | Use IAM policies with `ssm:StartSession` scoped by tag condition keys |
| VPC endpoint in wrong VPC / peered VPC | Instance inVpc-A, endpoint in Vpc-B | Interface endpoints are VPC-scoped; PrivateLink doesn't traverse VPC peering by default for endpoint DNS resolution the same way | Endpoint must exist in the same VPC as the instance (or reachable via endpoint-specific sharing/Resource Access Manager) |

---
