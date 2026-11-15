# AWS Monitoring with CloudWatch (EC2 VM)

## 1. Key Points

- **Amazon CloudWatch** is AWS's native monitoring service: metrics, logs, alarms, dashboards, events.
- **Basic monitoring (default, free)** for EC2: metrics every **5 minutes** (CPU, network, disk I/O, status checks).
- **Detailed monitoring (paid)**: metrics every **1 minute**. Enable per instance.
- **EC2 does NOT send memory, disk-space, or application logs by default.** The hypervisor can't see inside the OS. You need the **CloudWatch Agent** for:
  - Memory usage, swap
  - Disk used %, inodes
  - Custom logs (`/var/log/messages`, `/var/log/syslog`, nginx, app logs)
  - Custom metrics (StatsD / collectd)
- **Permissions are given to the VM through an IAM Role** (Instance Profile) – never store access keys on the VM.
- **Alarms** watch a metric and trigger actions: SNS notification, Auto Scaling, EC2 reboot/recover/stop.
- **Log Groups / Log Streams**: logs are stored in a *log group* (e.g. `/ec2/webapp`) with one *stream* per instance.
- **Retention**: log groups default to *never expire* – set a retention (e.g. 7/30 days) to control cost.
- Related services:
  | Service | Purpose |
  |---|---|
  | CloudWatch | Metrics, logs, alarms, dashboards |
  | CloudTrail | API call auditing (who did what) |
  | AWS Config | Resource configuration history/compliance |
  | X-Ray | Distributed tracing |
  | SNS | Send alarm notifications (email/SMS) |
  | Systems Manager (SSM) | Install/configure agent, run commands, Parameter Store |

---

## 2. Permissions Needed

### 2.1 For the EC2 VM (IAM Role / Instance Profile)

Attach this AWS managed policy to the role:

- **`CloudWatchAgentServerPolicy`** – lets the agent push metrics and logs.

Optional (recommended if you use SSM to install/configure the agent):

- **`AmazonSSMManagedInstanceCore`** – lets Systems Manager manage the instance.
- **`CloudWatchAgentAdminPolicy`** – only if the agent must write its config to SSM Parameter Store.

**Key actions inside `CloudWatchAgentServerPolicy`:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "ec2:DescribeVolumes",
        "ec2:DescribeTags",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams",
        "logs:DescribeLogGroups",
        "logs:CreateLogStream",
        "logs:CreateLogGroup"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameter"],
      "Resource": "arn:aws:ssm:*:*:parameter/AmazonCloudWatch-*"
    }
  ]
}
```

**Trust policy for the role (so EC2 can assume it):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 2.2 For the human user / admin (who sets things up)

| Task | Permissions needed |
|---|---|
| Create the IAM role | `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:CreateInstanceProfile`, `iam:AddRoleToInstanceProfile`, `iam:PassRole` |
| Attach role to EC2 | `ec2:AssociateIamInstanceProfile`, `iam:PassRole` |
| Enable detailed monitoring | `ec2:MonitorInstances` |
| View metrics/logs/dashboards | `cloudwatch:GetMetricData`, `cloudwatch:ListMetrics`, `logs:GetLogEvents`, `logs:FilterLogEvents` (or managed `CloudWatchReadOnlyAccess`) |
| Create alarms/dashboards | `cloudwatch:PutMetricAlarm`, `cloudwatch:PutDashboard` (or `CloudWatchFullAccess`) |
| Create SNS topic/subscription | `sns:CreateTopic`, `sns:Subscribe` |

### 2.3 Network requirement

The VM must reach CloudWatch endpoints:

- Public subnet with Internet Gateway, **or** NAT Gateway, **or**
- VPC Interface Endpoints for `monitoring` (metrics) and `logs` (CloudWatch Logs).
- Security group **outbound** HTTPS (443) must be allowed.

---

## 3. Steps to Enable Monitoring

### Step 1 – Create IAM Role for EC2

**Console:**
1. IAM → **Roles** → **Create role**
2. Trusted entity: **AWS service** → **EC2**
3. Attach policies: `CloudWatchAgentServerPolicy` (+ `AmazonSSMManagedInstanceCore`)
4. Name it e.g. `EC2-CloudWatch-Role` → Create

**CLI:**
```bash
aws iam create-role --role-name EC2-CloudWatch-Role \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy --role-name EC2-CloudWatch-Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy --role-name EC2-CloudWatch-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name EC2-CloudWatch-Role
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-CloudWatch-Role --role-name EC2-CloudWatch-Role
```

### Step 2 – Attach the Role to the EC2 Instance

**Console:** EC2 → select instance → **Actions → Security → Modify IAM role** → choose `EC2-CloudWatch-Role` → Save.

**CLI:**
```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-0123456789abcdef0 \
  --iam-instance-profile Name=EC2-CloudWatch-Role
```

### Step 3 – Enable Detailed Monitoring (optional, 1-min metrics)

**Console:** EC2 → instance → **Actions → Monitor and troubleshoot → Manage detailed monitoring** → Enable.

**CLI:**
```bash
aws ec2 monitor-instances --instance-ids i-0123456789abcdef0
```

### Step 4 – Install the CloudWatch Agent on the VM

**Amazon Linux 2 / 2023:**
```bash
sudo yum install -y amazon-cloudwatch-agent
```

**Ubuntu / Debian:**
```bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

**RHEL / CentOS:**
```bash
sudo yum install -y https://amazoncloudwatch-agent.s3.amazonaws.com/redhat/amd64/latest/amazon-cloudwatch-agent.rpm
```

**Via Systems Manager (no SSH):**
```bash
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets "Key=instanceids,Values=i-0123456789abcdef0" \
  --parameters '{"action":["Install"],"name":["AmazonCloudWatchAgent"]}'
```

### Step 5 – Create the Agent Configuration

Use the wizard:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Or write the config manually at `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "mem":  { "measurement": ["mem_used_percent"] },
      "swap": { "measurement": ["swap_used_percent"] },
      "disk": {
        "measurement": ["used_percent"],
        "resources": ["*"]
      },
      "cpu": {
        "measurement": ["cpu_usage_user", "cpu_usage_system"],
        "totalcpu": true
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/webapp/webapp.log",
            "log_group_name": "/ec2/webapp",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 7
          },
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/ec2/system",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

### Step 6 – Start the Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

Check status:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
sudo systemctl status amazon-cloudwatch-agent
```

Agent logs (for troubleshooting):
```bash
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### Step 7 – Verify in CloudWatch

- **Metrics**: CloudWatch → Metrics → All metrics → **CWAgent** namespace (memory/disk) and **AWS/EC2** namespace (CPU, network).
- **Logs**: CloudWatch → Logs → **Log groups** → `/ec2/webapp`.

### Step 8 – Create an SNS Topic (for alarm notifications)

```bash
aws sns create-topic --name ec2-alerts
aws sns subscribe --topic-arn arn:aws:sns:ap-south-1:<ACCOUNT_ID>:ec2-alerts \
  --protocol email --notification-endpoint you@example.com
```
Confirm the subscription from your email.

### Step 9 – Create a CloudWatch Alarm

Example: CPU > 80% for 5 minutes.

**Console:** CloudWatch → Alarms → **Create alarm** → Select metric → EC2 → Per-Instance → `CPUUtilization` → threshold 80 → choose SNS topic → Create.

**CLI:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-i-0123456789abcdef0" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --alarm-actions arn:aws:sns:ap-south-1:<ACCOUNT_ID>:ec2-alerts
```

Memory alarm (needs agent):
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "HighMemory-i-0123456789abcdef0" \
  --metric-name mem_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --alarm-actions arn:aws:sns:ap-south-1:<ACCOUNT_ID>:ec2-alerts
```

### Step 10 – Build a Dashboard

CloudWatch → **Dashboards** → **Create dashboard** → add widgets (line chart for CPU, memory, disk; log insights widget for errors).

---

## 4. Useful CloudWatch Logs Insights Queries

Errors in the last hour:
```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50
```

Count errors per 5 minutes:
```
filter @message like /ERROR/
| stats count() by bin(5m)
```

---

## 5. Common Troubleshooting

| Problem | Likely cause / fix |
|---|---|
| No memory/disk metrics | CloudWatch Agent not installed/running, or config missing `mem`/`disk` |
| `AccessDenied` / `UnauthorizedOperation` in agent log | IAM role not attached or missing `CloudWatchAgentServerPolicy` |
| Agent can't connect / timeout | No internet/NAT/VPC endpoint, or SG outbound 443 blocked |
| Logs not appearing | Wrong `file_path`, file unreadable by agent user, or agent not restarted after config change |
| Alarm stuck in `INSUFFICIENT_DATA` | Wrong namespace/dimensions, or metric not being published |
| High CloudWatch bill | Too many custom metrics, no log retention, 1-sec high-resolution metrics |

---

## 6. Best Practices

- Use **IAM roles**, never access keys, on VMs.
- Set **log retention** on every log group.
- Use **SSM Parameter Store** to hold one agent config and roll it out to many instances.
- Alarm on **memory and disk**, not only CPU.
- Use **composite alarms** to reduce noise.
- Tag instances (`Env`, `App`, `Owner`) and use tags in dashboards/alarms.
- Combine with **CloudTrail** (auditing) and **AWS Config** (compliance) for full observability.
- Manage everything as code (Terraform / CloudFormation / Ansible) so monitoring is repeatable.

---

## 7. Quick Checklist

- [ ] IAM role with `CloudWatchAgentServerPolicy` created
- [ ] Role attached to EC2 instance
- [ ] (Optional) Detailed monitoring enabled
- [ ] CloudWatch Agent installed
- [ ] Agent config created and started
- [ ] Metrics visible in `CWAgent` namespace
- [ ] Logs visible in log group
- [ ] SNS topic + email subscription confirmed
- [ ] Alarms created (CPU, memory, disk, status check)
- [ ] Dashboard created
- [ ] Log retention set
