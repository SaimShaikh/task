# AWS Hands-on Lab: Configure EBS Encryption Using AWS KMS

## Lab Overview

This lab teaches you how to secure Amazon EBS volumes using AWS Key
Management Service (AWS KMS). You will create a Customer Managed Key
(CMK), launch an encrypted EC2 instance, verify encryption, migrate an
existing unencrypted volume, enable EBS Encryption by Default, and
validate the configuration using both the AWS Console and AWS CLI.

------------------------------------------------------------------------

# Architecture

``` text
                    AWS KMS (CMK)
                alias/ebs-prod-key
                        │
                        ▼
              Encrypted Amazon EBS
                        │
                        ▼
                 Amazon EC2 Instance
                        │
                        ▼
              Read / Write Application Data
```

------------------------------------------------------------------------

# Learning Objectives

After completing this lab you will be able to:

-   Create a Customer Managed KMS Key
-   Understand AWS Managed Keys vs Customer Managed Keys
-   Launch an EC2 instance with encrypted EBS storage
-   Verify encryption using the Console and CLI
-   Encrypt an existing unencrypted EBS volume
-   Enable EBS Encryption by Default
-   Troubleshoot common encryption issues

------------------------------------------------------------------------

# Prerequisites

-   AWS Account
-   IAM user with EC2 and KMS permissions
-   Existing VPC
-   Existing Subnet
-   Existing Security Group
-   Existing EC2 Key Pair
-   AWS CLI configured (optional)

Recommended IAM permissions:

-   EC2
-   KMS
-   IAM (if creating roles)
-   CloudWatch (optional)

------------------------------------------------------------------------

# Part 0 -- Verify AWS Region

Confirm the AWS Region (example: ap-south-1).

> KMS keys are Regional. The EC2 instance, EBS volume and KMS key must
> all exist in the same Region.

------------------------------------------------------------------------

# Part 1 -- Create a Customer Managed KMS Key

1.  AWS Console → **Key Management Service (KMS)**.
2.  Select **Customer managed keys**.
3.  <img width="2780" height="1309" alt="image" src="https://github.com/user-attachments/assets/022d1a9a-5e0f-4ae5-b6d1-9e2cacd4b884" />

4.  Click **Create key**.
5.  Choose:
    -   Key Type: **Symmetric**
    -   Key Usage: **Encrypt and Decrypt**
6.  Click **Next**.
7.  Configure:
    -   Alias: `ebs-prod-key`
    -   Description: Customer Managed Key for EBS Encryption.
8.  Select **Key Administrators**.
9.  Select **Key Users**.
10.  Review the key policy.
11. Click **Finish**.
12. Open the key and enable **Automatic Key Rotation**.
13. Verify the key state is **Enabled**.

## AWS Managed Key vs Customer Managed Key

  AWS Managed       Customer Managed
  ----------------- ---------------------------
  alias/aws/ebs     alias/ebs-prod-key
  Managed by AWS    Managed by You
  Limited control   Full control
  Basic policy      Custom IAM & Key Policies

------------------------------------------------------------------------

# Part 2 -- Launch EC2 with Encrypted EBS

1.  Open **EC2 → Launch Instance**.
2.  Configure:
    -   Name
    -   Amazon Linux 2023
    -   Instance Type
    -   Key Pair
3.  Select your VPC, Subnet and Security Group.
4.  Record the **Availability Zone**.
5.  Under **Storage**:
    -   Volume Type: gp3
    -   Enable **Encrypted**
    -   Select KMS Key: `alias/ebs-prod-key`
  
<img width="2157" height="1036" alt="image" src="https://github.com/user-attachments/assets/fbd44745-9ec7-4e99-bd08-44794340efb0" />

6.  Launch the instance.

------------------------------------------------------------------------

# Part 3 -- Verify Encryption

## AWS Console

EC2 → Volumes

Verify:

-   Encrypted = Yes
-   KMS Key = alias/ebs-prod-key
<img width="3212" height="1829" alt="image" src="https://github.com/user-attachments/assets/b38b0464-c473-41e2-89a4-2162d0f1b3d8" />

## AWS CLI

Find Volume ID:

``` bash
aws ec2 describe-instances \
--instance-ids <instance-id> \
--query "Reservations[].Instances[].BlockDeviceMappings[].Ebs.VolumeId"
```

Verify encryption:

``` bash
aws ec2 describe-volumes \
--volume-ids <volume-id> \
--query "Volumes[].Encrypted"
```

Expected output:

``` text
true
```
<img width="3285" height="1332" alt="Screenshot 2026-07-27 at 11 39 46 AM" src="https://github.com/user-attachments/assets/aa7bf55e-dc9b-4082-a46d-6c5b38e7d14d" />

Verify KMS Key:

``` bash
aws ec2 describe-volumes \
--volume-ids <volume-id> \
--query "Volumes[].KmsKeyId"
```

------------------------------------------------------------------------

# Part 4 -- Encrypt an Existing Unencrypted Volume

> Existing EBS volumes cannot be encrypted directly.

Workflow:

``` text
Volume
  ↓
Snapshot
  ↓
Copy Snapshot (Enable Encryption)
  ↓
Encrypted Snapshot
  ↓
Create Encrypted Volume
  ↓
Detach Old Volume
  ↓
Attach New Volume
```

Steps:

1.  Create a snapshot.
2.  Wait for completion.
3.  Copy the snapshot.
4.  Enable **Encrypt this Snapshot**.
5.  Select `alias/ebs-prod-key`.
6.  Create an encrypted volume from the copied snapshot.
7.  Ensure the new volume is in the **same Availability Zone**.
8.  Stop the EC2 instance.
9.  Detach the old volume.
10. Attach the encrypted volume using the same device name.
11. Start the EC2 instance.
12. Verify boot:

``` bash
hostname
lsblk
df -h
```

------------------------------------------------------------------------

# Part 5 -- Enable EBS Encryption by Default

EC2 → Settings → EBS Encryption.

Enable:

-   Always encrypt new EBS volumes.
-   Choose `alias/ebs-prod-key`.

Verify:

``` bash
aws ec2 get-ebs-encryption-by-default
```

``` bash
aws ec2 get-ebs-default-kms-key-id
```

Launch a new EC2 instance and confirm the root volume is automatically
encrypted.

------------------------------------------------------------------------

# Troubleshooting

  Issue                 Resolution
  --------------------- -------------------------------------------
  AccessDenied          Check IAM/KMS permissions
  InvalidKeyUsage       Use a Symmetric KMS key
  Volume in wrong AZ    Create the volume in the same AZ
  Instance won't boot   Attach using the correct root device name
  Key Disabled          Enable the KMS key

------------------------------------------------------------------------

# Best Practices

-   Use Customer Managed Keys for production.
-   Enable automatic key rotation.
-   Follow the principle of least privilege.
-   Enable EBS Encryption by Default.
-   Take snapshots before replacing production root volumes.
-   Do not schedule deletion of KMS keys protecting active volumes.

------------------------------------------------------------------------

