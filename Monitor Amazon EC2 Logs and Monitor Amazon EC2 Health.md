# AWS CloudWatch Monitoring Lab — Complete Step-by-Step Guide

> **Goal:** Configure Amazon CloudWatch to monitor an Amazon Linux EC2 instance end-to-end — collecting default EC2 metrics, custom OS metrics (CPU, Memory, Disk) via the CloudWatch Agent, Linux log files, building a dashboard, and configuring alarms with SNS email notifications.

---

## Lab Architecture

```
              +-----------------------------+
              |        EC2 Instance         |
              |-----------------------------|
              | CloudWatch Agent            |
              |  - CPU                      |
              |  - Memory                   |
              |  - Disk                     |
              |  - Logs                     |
              +-----------------------------+
                            |
                  PutMetricData / PutLogEvents
                            |
                            v
              +-----------------------------+
              |      Amazon CloudWatch      |
              |-----------------------------|
              | - Metrics                   |
              | - Logs                      |
              | - Dashboard                 |
              | - Alarms                    |
              +-----------------------------+
                            |
                       Alarm Trigger
                            |
                            v
              +-----------------------------+
              |         Amazon SNS          |
              +-----------------------------+
                            |
                    Email Notification
                            |
                            v
                       Administrator
```

---

## Prerequisites

- AWS Account with console access
- An Amazon Linux EC2 instance (running)
- Internet connectivity from the instance (or the required VPC endpoints for CloudWatch/SNS)
- AWS CLI installed and configured on the instance
- `sudo` access on the instance

---

## Step 1 — Create and Attach IAM Role

1. Open **IAM → Roles → Create role**.
2. Select trusted entity type: **AWS Service**.
3. Choose use case: **EC2**.
4. Attach the permissions policy:
   - `CloudWatchAgentServerPolicy`
5. Name the role:
   - `EC2-CloudWatch-Role`
6. Click **Create role**.
7. Go to **EC2 → Instances → select your instance → Actions → Security → Modify IAM Role**.
8. Attach `EC2-CloudWatch-Role` to the instance.
9. Confirm/save.

---

## Step 2 — Install the CloudWatch Agent

Install the agent on the EC2 instance:

```bash
sudo yum install amazon-cloudwatch-agent -y
```

Verify the installation:

```bash
rpm -qa | grep amazon-cloudwatch-agent
```

---

## Step 3 — Generate the Agent Configuration (Wizard)

Run the configuration wizard:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Answer the wizard prompts as follows:

| Prompt | Selection |
|---|---|
| Operating system | Linux |
| Platform | EC2 |
| Run as user | root |
| StatsD | No |
| **CollectD** — *"WARNING: CollectD must be installed or the Agent will fail to start"* | **No** (recommended — see note below) |
| Collect host metrics (CPU, memory, etc.) | Yes |
| CPU metrics per core | Yes |
| Add EC2 dimensions (ImageId, InstanceId, InstanceType, AutoScalingGroupName) | Yes |
| Aggregate EC2 dimensions (InstanceId) | No |
| High resolution metrics interval | 4 → 60s |
| Default metrics config | 2 → Standard |
| Import existing CloudWatch Log Agent config | No |
| Collect log files | Yes |

> **Important — CollectD prompt:** The wizard warns that CollectD must already be installed on the host or the agent will fail to start. If you answer **"yes"** here (as in this lab), the wizard adds a `collectd` block under `metrics_collected`, and the agent will fail with a `socket_listener` / `types.db` error when you later run `fetch-config` (see the note in Step 4). Answering **"no"** avoids this entirely. If you do answer "yes," Step 4 below shows how to resolve it.

For the log files to monitor, add both of the following **one at a time**, using the **actual file path on disk** for each (not a CloudWatch log group path):

| Log file path | Log group name (default) | Log group class | Log stream name (default) | Retention |
|---|---|---|---|---|
| `/var/log/messages` | `messages` | STANDARD | `{instance_id}` | 30 days (option 7) |
| `/var/log/secure` | `secure` | STANDARD | `{instance_id}` | 30 days (option 7) |

> **Common mistake:** When prompted *"Do you want to specify any additional log files to monitor?"*, answer **1 (yes)** first, then enter the file path. Typing a path directly (e.g. `/aws/ec2/secure`) at the yes/no prompt is rejected as invalid, and typing a CloudWatch-style path (`/aws/ec2/secure`) instead of the real file path (`/var/log/secure`) will get accepted by the wizard but is wrong — the agent needs the actual log file location on the instance. Double-check this field says `/var/log/secure` before continuing.

After finishing the log file prompts, decline X-Ray traces (No) and decline storing the config in SSM Parameter Store (No, unless you want to).

This generates a configuration file at:

```
/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

---

## Step 4 — Review/Edit and Verify the Generated Configuration

Open the generated configuration file to review or edit it:

```bash
sudo vi /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

**If you answered "yes" to the CollectD prompt in Step 3**, your generated file will include a `collectd` entry under `metrics_collected`:

```json
"collectd": {
	"metrics_aggregation_interval": 60
}
```

This entry tells the agent to also listen for metrics from an external CollectD daemon over a socket. Since CollectD is not actually installed on this instance, attempting to load this configuration fails with an error referencing `/usr/share/collectd/types.db` (shown and resolved in Step 5).

**Fix:** Remove the entire `collectd` block from `metrics_collected`. If editing in place doesn't take effect cleanly, the more reliable approach (as used in this lab) is:

```bash
sudo rm /opt/aws/amazon-cloudwatch-agent/bin/config.json
sudo vi /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

Recreate the file with the `collectd` block removed. Also fix the second log file path here if it was entered as `/aws/ec2/secure` in Step 3 — it must be the real file path, `/var/log/secure`.

Below is the **exact, final, verified `config.json`** confirmed working in this lab (its `collectd`-free state is confirmed by the agent's own log output in Step 7, which lists only `telegraf_cpu`, `telegraf_disk`, `telegraf_diskio`, `telegraf_mem`, and `telegraf_swap` as receivers — no CollectD/socket listener):

```json
{
	"agent": {
		"metrics_collection_interval": 60,
		"run_as_user": "root"
	},
	"logs": {
		"logs_collected": {
			"files": {
				"collect_list": [
					{
						"file_path": "/var/log/messages",
						"log_group_class": "STANDARD",
						"log_group_name": "messages",
						"log_stream_name": "{instance_id}",
						"retention_in_days": 30
					},
					{
						"file_path": "/var/log/secure",
						"log_group_class": "STANDARD",
						"log_group_name": "secure",
						"log_stream_name": "{instance_id}",
						"retention_in_days": 30
					}
				]
			}
		}
	},
	"metrics": {
		"append_dimensions": {
			"AutoScalingGroupName": "${aws:AutoScalingGroupName}",
			"ImageId": "${aws:ImageId}",
			"InstanceId": "${aws:InstanceId}",
			"InstanceType": "${aws:InstanceType}"
		},
		"metrics_collected": {
			"cpu": {
				"measurement": [
					"cpu_usage_idle",
					"cpu_usage_iowait",
					"cpu_usage_user",
					"cpu_usage_system"
				],
				"metrics_collection_interval": 60,
				"resources": [
					"*"
				],
				"totalcpu": false
			},
			"disk": {
				"measurement": [
					"used_percent",
					"inodes_free"
				],
				"metrics_collection_interval": 60,
				"resources": [
					"*"
				]
			},
			"diskio": {
				"measurement": [
					"io_time"
				],
				"metrics_collection_interval": 60,
				"resources": [
					"*"
				]
			},
			"mem": {
				"measurement": [
					"mem_used_percent"
				],
				"metrics_collection_interval": 60
			},
			"swap": {
				"measurement": [
					"swap_used_percent"
				],
				"metrics_collection_interval": 60
			}
		}
	}
}
```

Save the file after editing.

---

## Step 5 — Load the Configuration

Apply the configuration to the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

**If your `config.json` still contains the `collectd` block** (from answering "yes" to CollectD in the wizard), this command fails during the second validation phase with:

```
Configuration validation second phase failed
======== Error Log ========
Error running agent: Error loading config file .../amazon-cloudwatch-agent.toml: error parsing socket_listener, open /usr/share/collectd/types.db: no such file or directory
```

**Fix:** Remove the `config.json`, recreate it without the `collectd` block (matching Step 4), and rerun the command above. You should now see:

```
Configuration validation first phase succeeded
...
Configuration validation second phase succeeded
Configuration validation succeeded
```

If the agent wasn't already running, you'll also see a message like `amazon-cloudwatch-agent has already been stopped` — this is expected and not an error; start it explicitly in Step 6.

---

## Step 6 — Start, Enable, and Verify the Agent

Start and enable the agent so it runs on boot:

```bash
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
sudo systemctl restart amazon-cloudwatch-agent
```

Check the service status:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

Check the agent's own status output:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-m ec2 \
-a status
```

Expected output:

```json
{
  "status": "running",
  "configstatus": "configured"
}
```

---

## Step 7 — Troubleshoot Using Agent Logs

Inspect the agent's internal log to confirm there are no errors:

```bash
sudo tail -100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

Confirm the log shows:

- No `AccessDenied` errors
- No IAM permission errors
- No `socket_listener` / CollectD errors
- Agent started successfully — `Starting AmazonCloudWatchAgent ...`
- `Everything is ready. Begin running and processing data.`
- The metrics receivers list only includes the plugins you actually configured — for this lab: `telegraf_cpu`, `telegraf_disk`, `telegraf_diskio`, `telegraf_mem`, `telegraf_swap` (no `collectd`/socket listener receiver)
- The logs plugin picked up both configured files: `start logs plugin file paths [/var/log/messages /var/log/secure]`
- EC2 tagger initialized successfully: `ec2tagger: Initial retrieval of tags succeeded`

---

## Step 8 — Verify Custom Metrics (CWAgent Namespace)

**Using the AWS CLI**, confirm metrics are being published:

```bash
aws cloudwatch list-metrics \
--namespace CWAgent \
--region eu-north-1
```

Confirm the namespace returned is:

```
CWAgent
```

**Using the CloudWatch Console:**

Navigate to **CloudWatch → Metrics → All metrics → CWAgent**.

Confirm the following metrics are present:

- `cpu_usage_idle`
- `cpu_usage_system`
- `cpu_usage_user`
- `cpu_usage_iowait`
- `mem_used_percent`
- `disk_used_percent`
- `disk_inodes_free`
- `swap_used_percent`
- `io_time`

---

## Step 9 — Verify Log Collection

Navigate to **CloudWatch → Logs → Log groups**.

Confirm the following log groups exist (names may include a prefix such as `/aws/ec2/`):

- `messages` (from `/var/log/messages`)
- `secure` (from `/var/log/secure`)

Confirm log retention is set to:

```
30 Days
```

---

## Step 10 — Create the CloudWatch Dashboard

Navigate to **CloudWatch → Dashboards → Create dashboard**.

Name the dashboard:

```
EC2-Monitoring-Dashboard
```

Add the following widgets:

- `CPUUtilization`
- `mem_used_percent`
- `disk_used_percent`
- `NetworkIn`
- `NetworkOut`
- `StatusCheckFailed`

Save the dashboard.

---

## Step 11 — Create the SNS Topic and Subscription

Navigate to **SNS → Topics → Create topic**.

Name the topic:

```
CloudWatch-Alerts
```

Create a subscription:

- Protocol: **Email**
- Endpoint: your email address

Confirm the subscription — either by clicking the confirmation link in the email, or via CLI:

```bash
aws sns confirm-subscription \
--topic-arn arn:aws:sns:eu-north-1:<ACCOUNT_ID>:CloudWatch-Alerts \
--token "<TOKEN>" \
--region eu-north-1
```

Verify the subscription is confirmed:

```bash
aws sns list-subscriptions-by-topic \
--topic-arn arn:aws:sns:eu-north-1:<ACCOUNT_ID>:CloudWatch-Alerts \
--region eu-north-1
```
<img width="1680" height="1050" alt="Screenshot 2026-07-27 at 3 34 35 PM" src="https://github.com/user-attachments/assets/b7a81218-c8cf-4d68-ae03-6f29e9617a05" />

---

## Step 12 — Create the CPU Alarm

In **CloudWatch → Alarms → Create alarm**, select the metric:

- Namespace: `AWS/EC2`
- Metric: `CPUUtilization`

Configure:

- Statistic: **Average**
- Period: **5 minutes**
- Threshold type: **Static**
- Condition: **Greater than**
- Threshold: **5** (for lab testing) or **80** (for production)

Set the notification action to the existing SNS topic:

```
CloudWatch-Alerts
```

Alarm name:

```
EC2-CPU-High
```

Click **Create alarm**.

---

## Step 13 — Create the Memory Alarm

In **CloudWatch → Alarms → Create alarm**, select the metric:

- Namespace: `CWAgent`
- Metric: `mem_used_percent`

Configure:

- Statistic: **Average**
- Period: **5 minutes**
- Condition: **Greater than**
- Threshold: **80** (or **5** for quick testing)

Set the notification action to:

```
CloudWatch-Alerts
```

Alarm name:

```
EC2-Memory-High
```

Click **Create alarm**.

---

## Step 14 — Create the Disk Alarm

In **CloudWatch → Alarms → Create alarm**, select the metric:

- Namespace: `CWAgent`
- Dimensions available: `ImageId`, `InstanceId`, `InstanceType`, `device`, `fstype`, `path`
- Metric: `disk_used_percent`

Select the row corresponding to the **root filesystem**:

| Path | Filesystem | Action |
|---|---|---|
| `/` | `xfs` | ✅ Select this one |

Do **not** select these other paths that also appear in the metric list:

- `/run`
- `/tmp`
- `/dev`
- `/dev/shm`
- `/boot/efi`
- `/run/user/1000`

(These are temporary or special filesystems and are not what is typically alarmed on.)

Steps:

1. Tick the checkbox for the row with **Path: `/`**, **Filesystem: `xfs`**.
2. Click **Select single metric**.
3. Configure:
   - Statistic: **Average**
   - Period: **5 minutes**
   - Threshold type: **Static**
   - Condition: **Greater than**
   - Threshold: **80** (or **5** for quick testing)
4. Click **Next**.
5. Set the notification action to the existing SNS topic:
   ```
   CloudWatch-Alerts
   ```
6. Alarm name:
   ```
   EC2-Disk-High
   ```
7. Click **Create alarm**.

---

## Step 15 — Test the Alarms

Install the `stress` tool on the EC2 instance:

```bash
sudo yum install stress -y
```

Generate CPU load to trigger the CPU alarm:

```bash
stress --cpu 2 --timeout 300
```

Observe the alarm state transition in the CloudWatch console:

```
INSUFFICIENT_DATA → OK → ALARM
```

Confirm the SNS email notification is received in your inbox.

---
## Outputs
<img width="3288" height="1762" alt="image" src="https://github.com/user-attachments/assets/53ced4b2-9f73-4834-be9a-73377b1dcbeb" />

<img width="2656" height="1631" alt="image" src="https://github.com/user-attachments/assets/78d6952d-eb6c-4409-8330-317dfb6f30a4" />
<img width="2624" height="1629" alt="image" src="https://github.com/user-attachments/assets/d80f6247-0058-4c5c-9596-7da3fa18b634" />

## Final Alarm Summary

| Alarm Name | Metric | Threshold |
|---|---|---|
| `EC2-CPU-High` | `CPUUtilization` | > 5% (lab) or > 80% (production) |
| `EC2-Memory-High` | `mem_used_percent` | > 80% |
| `EC2-Disk-High` | `disk_used_percent` (path `/`) | > 80% |

---

## Reference — All AWS CLI Commands Used

```bash
# Verify agent package installed
rpm -qa | grep amazon-cloudwatch-agent

# Load / apply the agent configuration
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s

# Start, enable, restart the agent service
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
sudo systemctl restart amazon-cloudwatch-agent
sudo systemctl status amazon-cloudwatch-agent

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-m ec2 \
-a status

# Review agent logs for troubleshooting
sudo tail -100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log

# Verify custom metrics are published
aws cloudwatch list-metrics \
--namespace CWAgent \
--region eu-north-1

# Confirm SNS subscription
aws sns confirm-subscription \
--topic-arn arn:aws:sns:eu-north-1:<ACCOUNT_ID>:CloudWatch-Alerts \
--token "<TOKEN>" \
--region eu-north-1

# Verify SNS subscription status
aws sns list-subscriptions-by-topic \
--topic-arn arn:aws:sns:eu-north-1:<ACCOUNT_ID>:CloudWatch-Alerts \
--region eu-north-1

# Install stress tool and generate CPU load for testing
sudo yum install stress -y
stress --cpu 2 --timeout 300
```

---

## Lab Outcome

By completing this lab, the following was successfully implemented:

- IAM Role for the CloudWatch Agent
- CloudWatch Agent installed, configured, and running
- Custom metrics published to the `CWAgent` namespace (CPU, Memory, Disk, Disk I/O, Swap)
- Log collection for `/var/log/messages` and `/var/log/secure` with 30-day retention
- A CloudWatch Dashboard (`EC2-Monitoring-Dashboard`) with CPU, Memory, Disk, Network, and Status Check widgets
- An SNS topic (`CloudWatch-Alerts`) with a confirmed email subscription
- Three CloudWatch Alarms (CPU, Memory, Disk) wired to the SNS topic
- Successful alarm testing using the `stress` tool, with a confirmed email notification

This results in a complete, end-to-end monitoring solution for an Amazon Linux EC2 instance.
