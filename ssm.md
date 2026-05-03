# AWS Systems Manager (SSM) — Teaching Guide

> **Audience:** Cloud / DevOps students  
> **Level:** Intermediate  
> **Prerequisite:** Basic AWS (EC2, IAM, CloudWatch)

---

## What is AWS Systems Manager?

AWS Systems Manager is a **central operations hub** for managing AWS and on-premises infrastructure at scale. Think of it as the *nervous system* of your cloud environment — it lets you **view, control, automate, and secure** your nodes from a single place, with no open ports or bastion hosts required.

### Key Benefits
- No SSH / RDP required — fully browser and API driven
- Works for EC2, on-premises servers, VMs, and edge devices
- Full audit trail of every action
- Integrated with IAM, CloudWatch, CloudTrail, and Config

### Core Concept — Managed Nodes
Any server (EC2 or on-prem) with the **SSM Agent** installed and an IAM Instance Profile attached becomes a *managed node*. Once managed, you can run all SSM tools against it.

```
EC2 Instance
  └── SSM Agent (installed & running)
  └── IAM Instance Profile (AmazonSSMManagedInstanceCore policy)
      └── Registered as Managed Node in SSM
```

---

## Category 1 — Node Tools

Node Tools help you **inventory, control, patch, and connect** to your managed instances.

---

### 1.1 Session Manager

**What it does:** Opens a secure interactive shell (bash / PowerShell) to any managed node — directly from the AWS Console or CLI — with zero open inbound ports.

**Why it matters:**
- No port 22 (SSH) or 3389 (RDP) needed on Security Groups
- No key pairs or bastion hosts required
- All session activity is logged to S3 and CloudWatch Logs

**Real-world use case:**  
A developer needs to debug a private EC2 instance inside a VPC with no internet gateway. Using Session Manager, they open a shell directly from the console without any network changes.

**Key CLI command:**
```bash
aws ssm start-session --target i-0123456789abcdef0
```

---

### 1.2 Run Command

**What it does:** Runs shell scripts or PowerShell commands across one or many managed instances — remotely, simultaneously, with no SSH.

**Why it matters:**
- Execute the same command on hundreds of servers at once
- Built-in rate control and error thresholds
- Results stored in S3 or CloudWatch Logs

**Real-world use case:**  
Restart the `nginx` service on all web servers tagged `Env=Production` without logging into each one.

**Key CLI command:**
```bash
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Env,Values=Production" \
  --parameters commands="sudo systemctl restart nginx"
```

---

### 1.3 Patch Manager

**What it does:** Automates OS and application patching across your entire fleet on a defined schedule.

**Core concepts:**
| Concept | Description |
|---|---|
| Patch Baseline | Rules defining which patches are approved/rejected |
| Patch Group | Tag-based grouping of instances for targeted patching |
| Maintenance Window | Scheduled time slot for patching to run |

**Real-world use case:**  
Auto-patch all Linux servers every Sunday at 2:00 AM, only applying Critical and High severity patches, and automatically rebooting if required.

---

### 1.4 Inventory

**What it does:** Collects and stores metadata from managed nodes — installed applications, OS version, network config, Windows services, running processes.

**Real-world use case:**  
An auditor asks: "Which servers are running Python 2.7?" Use Inventory + AWS Config aggregator to query across your entire fleet in seconds.

---

### 1.5 Compliance

**What it does:** Shows a compliance dashboard for patch status and State Manager associations across your fleet.

**Real-world use case:**  
Before a security audit, check that 100% of production servers are compliant with your approved patch baseline.

---

### 1.6 Fleet Manager

**What it does:** A GUI-based console to manage nodes — browse the file system, view logs, check running processes — without any terminal access.

**Real-world use case:**  
A Windows admin needs to check Event Viewer logs on a private server. Fleet Manager provides a browser-based UI without needing RDP.

---

### 1.7 Distributor

**What it does:** Packages software (agents, tools, custom apps) and distributes them to managed instances on demand or on a schedule.

**Real-world use case:**  
Deploy the CloudWatch unified agent to 500 new EC2 instances automatically when they join the fleet.

---

### 1.8 Hybrid Activations

**What it does:** Registers non-EC2 machines (on-premises servers, VMs, IoT devices) as SSM managed nodes.

**Real-world use case:**  
A company running 200 physical servers in a data center wants to use Patch Manager and Session Manager for them — Hybrid Activations makes this possible.

---

### 1.9 State Manager

**What it does:** Continuously enforces a desired configuration state on managed nodes by running SSM Documents on a schedule.

**Real-world use case:**  
Ensure the SSM agent is always running, updated, and correctly configured — even if someone manually stops it.

---

## Category 2 — Change Management Tools

Change Management Tools help you **plan, approve, document, and execute** changes to your infrastructure safely.

---

### 2.1 Documents (SSM Documents)

**What it does:** JSON or YAML files that define a set of actions — the building blocks for Run Command, Automation, State Manager, and Patch Manager.

**Document types:**
| Type | Used by |
|---|---|
| Command | Run Command, State Manager |
| Automation | Automation runbooks |
| Session | Session Manager preferences |
| Package | Distributor |

**Example — simple shell command document:**
```yaml
schemaVersion: "2.2"
description: "Restart Nginx"
mainSteps:
  - action: aws:runShellScript
    name: restartNginx
    inputs:
      runCommand:
        - sudo systemctl restart nginx
```

---

### 2.2 Automation

**What it does:** Runs multi-step operational runbooks across AWS resources — not just instances, but also S3, RDS, IAM, EC2 AMIs, and more.

**Why it matters:**
- Automates complex, multi-service workflows
- Can require approvals at specific steps
- Integrates with Change Calendar to block runs during frozen periods

**Real-world use case:**  
Golden AMI pipeline: Launch a base instance → Install updates → Run tests → Create AMI → Share to all accounts → Terminate instance. Fully automated, triggered on a schedule.

---

### 2.3 Maintenance Windows

**What it does:** Defines recurring time windows during which maintenance tasks (patching, automation, run commands) are allowed to execute.

**Key fields:**
- Schedule (cron or rate expression)
- Duration (hours)
- Cutoff (stop registering new tasks N hours before window closes)
- Targets (which instances)

**Real-world use case:**  
Run Patch Manager every first Sunday of the month, 1:00–5:00 AM, with a 1-hour cutoff — giving 4 hours of safe patching time.

---

### 2.4 Change Manager

**What it does:** An ITIL-style change request and approval workflow, integrated with Change Calendar and Automation.

**Workflow:**
```
Request Created → Approvers Notified → Approved/Rejected
      └── If Approved → Check Change Calendar → Run Automation Runbook
```

**Real-world use case:**  
Before doing any production database maintenance, require sign-off from a DBA and a security engineer via Change Manager — with full audit trail.

---

### 2.5 Change Calendar

**What it does:** Defines periods when changes are allowed (OPEN) or blocked (CLOSED) — integrated with Automation and Change Manager.

**Real-world use case:**  
Block all automated deployments and runbooks during the end-of-year financial reporting period (Dec 15 – Jan 3).

---

### 2.6 Quick Setup

**What it does:** A wizard that automatically configures common SSM features across multiple accounts and regions with best-practice defaults.

**Real-world use case:**  
A new AWS Organization wants to enable SSM host management, automatic patching, and resource data sync across 20 accounts in 3 regions — Quick Setup does it in minutes.

---

## Category 3 — Application Tools

Application Tools help you **configure, manage, and secure** your applications and their runtime settings.

---

### 3.1 Parameter Store

**What it does:** Secure, hierarchical key-value store for configuration data and secrets — accessible by Lambda, EC2, ECS, CodePipeline, and more.

**Parameter types:**
| Type | Use case |
|---|---|
| String | Non-sensitive config (e.g., region name) |
| StringList | Comma-separated values |
| SecureString | Secrets encrypted with KMS (DB passwords, API keys) |

**Hierarchy example:**
```
/myapp/prod/db/host        → String
/myapp/prod/db/port        → String
/myapp/prod/db/password    → SecureString (KMS encrypted)
/myapp/prod/feature/darkmode → String ("true"/"false")
```

**CLI example:**
```bash
# Write
aws ssm put-parameter --name "/prod/db/password" \
  --value "MySecretPass!" --type SecureString

# Read (in a Lambda or EC2 startup script)
aws ssm get-parameter --name "/prod/db/password" --with-decryption
```

---

### 3.2 AppConfig

**What it does:** Manages and safely deploys application configuration (feature flags, tuning parameters) independent of code deployments — with validation and rollback.

**Deployment strategies:**
- All at once
- Linear (e.g., 10% every 5 minutes)
- Exponential (canary rollout)

**Real-world use case:**  
Enable a new "dark mode" feature flag for 10% of users, monitor error rates for 10 minutes, then automatically roll out to 100% — or roll back if errors spike.

---

### 3.3 Application Manager

**What it does:** Groups all AWS resources belonging to an application (ECS services, EC2, Lambda, RDS, CloudFormation stacks) into a single operational view.

**Real-world use case:**  
For the "checkout-service" application, see all its alarms, recent runbooks, cost breakdown, OpsItems, and patch compliance — in one unified console.

---

## Category 4 — Operations Tools

Operations Tools give you **visibility, alerting, and incident response** across your entire AWS environment.

---

### 4.1 Explorer

**What it does:** A customizable multi-account, multi-region operations dashboard aggregating patch compliance, OpsItems, Config rule results, and EC2 health data.

**Real-world use case:**  
As an ops lead managing 15 AWS accounts, use Explorer to see at a glance: how many instances are non-compliant, how many open OpsItems exist, and which regions have failing Config rules.

---

### 4.2 OpsCenter

**What it does:** A central inbox for operational issues (called *OpsItems*) — automatically created from CloudWatch alarms, Config rules, and GuardDuty findings — with linked runbooks for remediation.

**OpsItem lifecycle:**
```
CloudWatch Alarm fires
    └── OpsItem auto-created
    └── On-call engineer investigates
    └── Runs linked Automation runbook
    └── OpsItem resolved + documented
```

**Real-world use case:**  
When a CloudWatch alarm detects high CPU on an RDS instance, an OpsItem is automatically created. The engineer opens it, sees related metrics, and clicks "Run runbook" to scale up the instance — all from one screen.

---

### 4.3 Incident Manager

**What it does:** Structured incident response — automated contact escalation, runbook execution during incidents, and post-incident analysis (PIR).

**Key concepts:**
| Concept | Description |
|---|---|
| Response Plan | Defines who to contact and which runbooks to run |
| Escalation Plan | Auto-escalates if first contact doesn't acknowledge |
| Engagement | Paging contacts via SMS, phone, or PagerDuty |
| Post-Incident Analysis | Root cause + timeline after resolution |

**Real-world use case:**  
A production API goes down at 3 AM. Incident Manager auto-pages the on-call engineer, starts a diagnostic runbook, opens a collaboration chat channel, and records every action — so the PIR writes itself.

---

### 4.4 CloudWatch Dashboard (in Explorer)

**What it does:** A pre-built CloudWatch metrics and alarms view — CPU, memory, disk, network — surfaced directly inside Systems Manager for your managed nodes.

---

## Quick Reference — Which Tool for Which Job?

| Scenario | Tool |
|---|---|
| SSH into a private instance safely | Session Manager |
| Run a script on 200 servers at once | Run Command |
| Patch all prod servers automatically | Patch Manager + Maintenance Windows |
| Store a DB password securely | Parameter Store (SecureString) |
| Deploy a feature flag gradually | AppConfig |
| Require approval before prod change | Change Manager |
| Block deployments during a freeze | Change Calendar |
| Multi-account compliance dashboard | Explorer |
| Respond to a production incident | Incident Manager |
| Manage on-premises servers with SSM | Hybrid Activations |

---

## Lab Exercise Ideas

1. **Lab 1 — First managed node:** Launch an EC2 instance with the SSM Instance Profile, connect via Session Manager, run a command with Run Command.

2. **Lab 2 — Patch Manager:** Create a patch baseline, create a maintenance window, patch a test instance and verify compliance.

3. **Lab 3 — Parameter Store:** Store a database connection string as a SecureString, retrieve it from a Lambda function without hardcoding credentials.

4. **Lab 4 — Automation runbook:** Write a custom SSM Document to stop, snapshot, and restart an EC2 instance — triggered via the Automation console.

5. **Lab 5 — Incident response:** Set up a CloudWatch alarm → OpsCenter OpsItem → linked runbook for remediation. Trigger the alarm and walk through the full response flow.

---

## Summary

AWS Systems Manager removes the need for bastion hosts, manual patching, and scattered configuration files. It provides a single, audited, automated control plane for your entire infrastructure — on AWS and on-premises.

| Category | Purpose |
|---|---|
| Node Tools | Connect, patch, inventory, and control instances |
| Change Management | Plan, approve, document, and safely execute changes |
| Application Tools | Manage config, secrets, and feature flags |
| Operations Tools | Monitor, alert, and respond to operational issues |

---

*Guide prepared for classroom use. For the latest service features, refer to the [AWS SSM Documentation](https://docs.aws.amazon.com/systems-manager/).*
