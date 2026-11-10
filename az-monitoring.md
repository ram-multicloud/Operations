# Azure Monitoring for Virtual Machines
### Enable monitoring, collect metrics and application logs, and build a dashboard

---

## 1. What is Azure Monitor?

**Azure Monitor** is Azure's built-in platform for collecting, analyzing, and acting on telemetry from your cloud resources. For a VM it gives you two kinds of data:

| Data type | What it is | Where it is stored | Examples |
|---|---|---|---|
| **Metrics** | Numeric time-series values, lightweight and near real-time | Azure Monitor Metrics database | CPU %, disk IOPS, network in/out |
| **Logs** | Text/event records, queried with KQL (Kusto Query Language) | Log Analytics workspace | Windows Event Logs, Linux Syslog, IIS logs, custom app logs |

### Key components

| Component | Purpose |
|---|---|
| **Platform (host) metrics** | Collected automatically for every VM, no setup. Measured from the hypervisor (CPU, disk, network). |
| **Azure Monitor Agent (AMA)** | Agent installed inside the VM to collect *guest OS* metrics, performance counters, and logs. Replaces the retired Log Analytics (MMA/OMS) agent. |
| **Data Collection Rule (DCR)** | Defines *what* to collect from the VM and *where* to send it. |
| **Log Analytics workspace** | The store where logs are sent and queried. |
| **VM Insights** | Ready-made monitoring experience (performance charts, process map, dependencies) built on AMA + DCR. |
| **Workbooks / Dashboards / Grafana** | Visualization options. |

### Architecture

```
 ┌─────────────────────────┐
 │  Azure VM               │
 │  ┌───────────────────┐  │      ┌──────────────────────┐
 │  │ Azure Monitor     │──┼─────▶│ Data Collection Rule │
 │  │ Agent (AMA)       │  │      └──────────┬───────────┘
 │  └───────────────────┘  │                 │
 │  App logs, Event Logs,  │                 ▼
 │  Syslog, Perf counters  │      ┌──────────────────────┐
 └─────────────────────────┘      │ Log Analytics        │
          │ Platform metrics      │ Workspace (KQL)      │
          ▼                       └──────────┬───────────┘
 ┌─────────────────────────┐                 │
 │ Azure Monitor Metrics   │                 ▼
 └──────────┬──────────────┘      ┌──────────────────────┐
            └────────────────────▶│ Dashboard / Workbook │
                                  │ Alerts               │
                                  └──────────────────────┘
```

---

## 2. Prerequisites

- An Azure subscription and a running VM (Windows or Linux)
- Permissions: **Contributor** (or **Monitoring Contributor** + **Virtual Machine Contributor**) on the VM and resource group
- A **Log Analytics workspace** (created below if you don't have one)
- Outbound connectivity from the VM to Azure Monitor endpoints (or Private Link / proxy configured)

---

## 3. Quick look: metrics with zero setup

Every VM already sends **host-level metrics**.

1. Azure Portal → **Virtual machines** → select your VM
2. Left menu → **Monitoring → Metrics**
3. Pick a metric (e.g. *Percentage CPU*), aggregation (Avg/Max), and time range

These are limited to host-level data. For memory, disk space, processes, and application logs you need the agent (next sections).

---

## 4. Enable monitoring for a VM

### Option A: Azure Portal (recommended for beginners): VM Insights

1. Go to **Virtual machines** → select your VM
2. Left menu → **Monitoring → Insights**
3. Click **Enable**
4. Choose or create a **Log Analytics workspace**
5. Choose or create a **Data Collection Rule** (the default enables performance counters and, optionally, the Map/dependency data)
6. Click **Configure / Enable** and wait a few minutes

Behind the scenes Azure will:
- Enable a **system-assigned managed identity** on the VM
- Install the **Azure Monitor Agent** extension
- Create a **DCR** and associate it with the VM

**Verify:** VM → **Extensions + applications** should list `AzureMonitorWindowsAgent` or `AzureMonitorLinuxAgent` with status *Provisioning succeeded*.

### Option B: Azure CLI

```bash
# Variables
RG="rg-monitoring-demo"
LOC="eastus"
VM="vm-demo-01"
LAW="law-monitoring-demo"

# 1. Create a Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group $RG \
  --workspace-name $LAW \
  --location $LOC

# 2. Enable system-assigned managed identity on the VM
az vm identity assign --resource-group $RG --name $VM

# 3. Install Azure Monitor Agent
# Linux:
az vm extension set \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --resource-group $RG \
  --vm-name $VM \
  --enable-auto-upgrade true

# Windows (use this instead on Windows VMs):
az vm extension set \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --resource-group $RG \
  --vm-name $VM \
  --enable-auto-upgrade true
```

> The agent alone collects nothing. You must also create and associate a **Data Collection Rule** (Section 5).

### Option C: Azure Policy (at scale)

For many VMs, assign built-in policy initiatives such as *"Enable Azure Monitor for VMs"* so that new VMs are onboarded automatically.

---

## 5. Collect VM metrics and application logs with a Data Collection Rule

### 5.1 Create the DCR in the portal

1. Search for **Monitor** → **Settings → Data Collection Rules** → **Create**
2. **Basics:** name, subscription, resource group, region, platform type (Windows / Linux)
3. **Resources:** add your VM(s)
4. **Collect and deliver:** click **Add data source**

Add the data sources you need:

| Data source | Use it for | Destination table |
|---|---|---|
| **Performance Counters** | CPU, memory, disk, network (guest OS) | `Perf` |
| **Windows Event Logs** | Application / System / Security events | `Event` |
| **Linux Syslog** | System and daemon messages | `Syslog` |
| **Custom Text Logs** | Your own application log files | `<YourName>_CL` |
| **IIS Logs** | Web server access logs (Windows) | `W3CIISLog` |

5. Set the **destination** to your Log Analytics workspace
6. **Review + create**

### 5.2 Example: collect application logs from a text file

Suppose your app writes logs to `/var/log/myapp/app.log` (Linux) or `C:\inetpub\logs\app\app.log` (Windows).

1. In the DCR → **Add data source** → **Custom Text Logs**
2. Set **File pattern**: `/var/log/myapp/*.log`
3. Set **Table name**: `MyAppLogs_CL`
4. Define the **record delimiter** (end of line by default) and the **transform** (optional KQL to parse fields)
5. Destination: your Log Analytics workspace

> Tip: Writing logs in **JSON, one entry per line**, makes parsing in KQL much easier.

### 5.3 Associate the DCR with the VM (CLI)

```bash
DCR_ID=$(az monitor data-collection rule show \
  --name dcr-vm-monitoring --resource-group $RG --query id -o tsv)

VM_ID=$(az vm show --name $VM --resource-group $RG --query id -o tsv)

az monitor data-collection rule association create \
  --name dcra-vm-demo \
  --rule-id $DCR_ID \
  --resource $VM_ID
```

### 5.4 Verify data is flowing

Wait about 5 to 10 minutes, then in the **Log Analytics workspace → Logs** run:

```kusto
Heartbeat
| where Computer == "vm-demo-01"
| summarize LastHeartbeat = max(TimeGenerated) by Computer
```

If you see a recent timestamp, the agent is reporting.

---

## 6. Useful KQL queries

### VM metrics

**CPU utilization over time**
```kusto
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize AvgCPU = avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart
```

**Available memory (Windows)**
```kusto
Perf
| where ObjectName == "Memory" and CounterName == "Available MBytes"
| summarize AvgFreeMB = avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart
```

**CPU using VM Insights table**
```kusto
InsightsMetrics
| where Namespace == "Processor" and Name == "UtilizationPercentage"
| summarize AvgCPU = avg(Val) by bin(TimeGenerated, 5m), Computer
| render timechart
```

**Free disk space %**
```kusto
InsightsMetrics
| where Namespace == "LogicalDisk" and Name == "FreeSpacePercentage"
| summarize AvgFree = avg(Val) by bin(TimeGenerated, 15m), Computer
| render timechart
```

### Application and system logs

**Windows application errors**
```kusto
Event
| where EventLog == "Application" and EventLevelName in ("Error", "Warning")
| project TimeGenerated, Computer, Source, EventID, RenderedDescription
| order by TimeGenerated desc
```

**Linux Syslog errors**
```kusto
Syslog
| where SeverityLevel in ("err", "crit", "alert", "emerg")
| project TimeGenerated, Computer, Facility, ProcessName, SyslogMessage
| order by TimeGenerated desc
```

**Custom application log search**
```kusto
MyAppLogs_CL
| where RawData contains "ERROR"
| project TimeGenerated, Computer, RawData
| order by TimeGenerated desc
```

**Error count trend**
```kusto
MyAppLogs_CL
| where RawData contains "ERROR"
| summarize Errors = count() by bin(TimeGenerated, 15m)
| render columnchart
```

---

## 7. Create a dashboard

There are three main options. Pick based on your needs.

| Option | Best for | Pros | Cons |
|---|---|---|---|
| **Azure Portal Dashboard** | Simple, shareable overview | Quick, pin charts/queries, RBAC sharing | Limited interactivity |
| **Azure Workbook** | Rich, interactive reports | Parameters, multiple tabs, mix metrics + logs | More setup |
| **Azure Managed Grafana** | Advanced / multi-source visualization | Powerful, many data sources | Extra service and cost |

### 7.1 Option 1: Azure Portal Dashboard

**Step 1: Create the dashboard**
1. Portal → **Dashboard** (top-left menu) → **Create** → **Custom**
2. Name it, e.g. `VM-Monitoring-Dashboard`
3. Choose subscription and resource group, then **Save**

**Step 2: Pin metric charts**
1. VM → **Monitoring → Metrics**
2. Build a chart (e.g. *Percentage CPU*, *Disk Read/Write Bytes*, *Network In/Out Total*)
3. Click **Save to dashboard → Pin to dashboard** and select your dashboard

**Step 3: Pin log query results**
1. Log Analytics workspace → **Logs**
2. Run a KQL query that produces a chart (use `render timechart` or `render columnchart`)
3. Click **Pin to** → **Azure dashboard**
4. Select your dashboard

**Step 4: Arrange and share**
1. Open the dashboard → **Edit** to resize and rearrange tiles
2. **Share** → publish to a resource group so others with access can view it

**Suggested dashboard layout**

| Row | Tiles |
|---|---|
| Health | VM availability (Heartbeat), CPU %, Memory available |
| Disk and network | Disk IOPS, Free space %, Network in/out |
| App logs | Error count trend, latest errors table, warnings by source |

### 7.2 Option 2: Azure Workbook (recommended for richer views)

1. VM → **Monitoring → Workbooks** (or Monitor → Workbooks)
2. Open a template such as **Performance Analysis** or **Virtual Machines** under *Virtual Machines* templates, or click **+ New**
3. Click **Add → Add query** (select *Logs*) and paste a KQL query
4. Choose visualization: **Time chart**, **Bar chart**, **Grid**, or **Tiles**
5. Add parameters (**Add → Add parameters**) such as **Time range**, **Subscription**, **Computer** so viewers can filter
6. Click **Done Editing** → **Save** → choose a resource group and name
7. Pin individual workbook items to a dashboard if needed (Pin icon on each item)

### 7.3 Option 3: Azure Managed Grafana

1. Create an **Azure Managed Grafana** instance (Portal → *Create a resource*)
2. Grant it **Monitoring Reader** on your subscription (done automatically in most cases)
3. Open Grafana → the **Azure Monitor** data source is preconfigured
4. Import a prebuilt dashboard (search Grafana for *Azure Virtual Machine*) or create panels with Azure Monitor metrics and Log Analytics (KQL) queries

---

## 8. Add alerts (recommended)

Dashboards are passive; alerts notify you.

1. VM → **Monitoring → Alerts** → **Create → Alert rule**
2. **Condition:** e.g. *Percentage CPU* > 85% for 5 minutes, or a log query such as errors > 10 in 15 minutes
3. **Action group:** email, SMS, Teams/webhook, or an automation runbook
4. Set severity and **Create**

**Log alert example query**
```kusto
MyAppLogs_CL
| where RawData contains "ERROR"
| summarize ErrorCount = count()
```
Condition: `ErrorCount > 10`, evaluated every 5 minutes over a 15-minute window.

---

## 9. Best practices and cost tips

- **Collect only what you need.** Log ingestion is billed per GB, so filter noisy logs with DCR transformations.
- **Use one workspace per environment** (e.g. dev / prod) to keep access control and cost clear.
- **Set data retention** on the workspace (default 30 days; longer costs more). Consider archive tiers for compliance.
- **Set a daily cap** on the workspace while testing to avoid surprise bills.
- **Use tags and consistent naming** for VMs, DCRs, and workspaces.
- **Use structured (JSON) application logs** for easier querying.
- **Secure access** with Azure RBAC (Monitoring Reader for viewers, Monitoring Contributor for editors).
- **Migrate off legacy agents.** The Log Analytics agent (MMA) is retired; use Azure Monitor Agent.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No data in workspace | DCR not associated with VM | Check **DCR → Resources**; re-create the association |
| Agent extension failed | Missing managed identity or no outbound access | Enable system-assigned identity; allow outbound HTTPS to Azure Monitor endpoints |
| `Heartbeat` empty | Agent not running or wrong region/workspace | Restart agent service; confirm DCR destination workspace |
| Custom logs missing | Wrong file pattern or permissions | Verify path and that the agent can read the file; check DCR table name ends with `_CL` |
| Memory/disk charts empty | Only host metrics enabled | Add Performance Counters in the DCR or enable VM Insights |
| Dashboard tiles show "No data" | Time range or scope mismatch | Check tile time range and the workspace/VM it targets |

**Check the agent on the VM**

```bash
# Linux
sudo systemctl status azuremonitoragent
sudo journalctl -u azuremonitoragent --since "30 minutes ago"
```

```powershell
# Windows
Get-Service AzureMonitorAgent
```

---

## 11. Summary checklist

- [ ] Create a **Log Analytics workspace**
- [ ] Enable **managed identity** and install **Azure Monitor Agent** on the VM
- [ ] Create a **Data Collection Rule** (performance counters, event logs/syslog, custom app logs)
- [ ] **Associate** the DCR with the VM
- [ ] **Verify** data with a `Heartbeat` query
- [ ] Build a **Dashboard or Workbook** with metric charts and KQL log tiles
- [ ] Add **alerts** for CPU, disk, and application errors
- [ ] Review **retention and cost** settings

---

## References

- Azure Monitor overview: https://learn.microsoft.com/azure/azure-monitor/overview
- Azure Monitor Agent: https://learn.microsoft.com/azure/azure-monitor/agents/azure-monitor-agent-overview
- VM Insights: https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-overview
- Data Collection Rules: https://learn.microsoft.com/azure/azure-monitor/essentials/data-collection-rule-overview
- KQL reference: https://learn.microsoft.com/azure/data-explorer/kusto/query/
- Azure Workbooks: https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-overview
