# Azure Dashboard for Dummy Web-App Logs: Step-by-Step

## Overview

```
generate_logs.py -> /var/log/webapp/webapp.log -> Azure Monitor Agent (AMA)
   -> Data Collection Rule (DCR) -> Log Analytics table (WebAppLogs_CL)
   -> KQL queries -> Azure Dashboard / Workbook
```

**Assumptions:** the script runs on an Azure Linux VM (or an Arc-enabled server) and you have permission to create resources in a subscription.

**Log line format** (produced by `generate_logs.py`):

```
2026-11-11T10:15:00  INFO     User login successful. UserID=1234
```

---

## Step 1: Start the log generator on the VM

```bash
sudo mkdir -p /var/log/webapp
sudo chmod 755 /var/log/webapp

# Run in background, 1 log line per second
sudo nohup python3 generate_logs.py -i 1 > /dev/null 2>&1 &

# Make sure the agent can read the file
sudo chmod 644 /var/log/webapp/webapp.log

# Verify logs are being written
tail -f /var/log/webapp/webapp.log
```

> On Ubuntu/Debian use `python3` (not `python`).

---

## Step 1b: Run the generator with cron (alternative to nohup)

Use this if you want the logs generated on a schedule instead of one long-running process. The script file on the VM is `/home/azureuser/generate_dummy_logs.py`.

### Check the script is the version with options

```bash
python3 generate_dummy_logs.py --help
```

You should see `-f`, `-i` and `-n`. If not, replace the file with the improved script. The first version ignores `-n` and loops forever, so every cron run would pile up.

### Test manually first

```bash
sudo python3 generate_dummy_logs.py -n 5 -i 0
sudo tail -5 /var/log/webapp/webapp.log
```

### Find the full paths (cron has a minimal environment)

```bash
which python3        # usually /usr/bin/python3
realpath generate_dummy_logs.py
```

### Add the cron job (root, because /var/log/webapp needs root)

```bash
sudo crontab -e
```

```cron
* * * * * /usr/bin/flock -n /tmp/generate_logs.lock /usr/bin/python3 /home/azureuser/generate_dummy_logs.py -n 30 -i 2 >> /tmp/generate_logs.cron.log 2>&1
```

This runs every minute and writes 30 lines (one every 2 seconds). `flock -n` skips a run if the previous one is still going, so runs never overlap.

Or add it without opening an editor:

```bash
(sudo crontab -l 2>/dev/null; echo "* * * * * /usr/bin/flock -n /tmp/generate_logs.lock /usr/bin/python3 /home/azureuser/generate_dummy_logs.py -n 30 -i 2 >> /tmp/generate_logs.cron.log 2>&1") | sudo crontab -
```

### Other schedules

```cron
*/5 * * * *  ... -n 100 -i 0        # every 5 minutes, 100 lines instantly
0 * * * *    ... -n 500 -i 0        # every hour
@reboot      ... -i 1               # start at boot and run continuously (no -n)
```

### Verify cron is working

```bash
sudo crontab -l                                  # list jobs
sudo tail -f /var/log/webapp/webapp.log          # watch new lines arrive
grep CRON /var/log/syslog | tail                 # cron execution log
cat /tmp/generate_logs.cron.log                  # script output and errors
```

### Make sure the Azure agent can read the file

```bash
sudo chmod 644 /var/log/webapp/webapp.log
```

### Stop or remove the cron job

```bash
sudo crontab -e              # delete the line
sudo crontab -r              # or remove all root cron jobs
sudo pkill -f generate_dummy_logs.py   # stop a run in progress
```

---

## Step 2: Create a Log Analytics workspace

1. Azure Portal -> **Log Analytics workspaces** -> **Create**
2. Choose subscription, resource group (e.g. `rg-logs`), name (e.g. `law-webapp`) and region
3. **Review + create**

---

## Step 3: Create the custom table and Data Collection Rule

1. Open the workspace -> **Tables** -> **Create** -> **New custom log** (text logs for Azure Monitor Agent)
2. Table name: `WebAppLogs` (Azure appends `_CL`, giving `WebAppLogs_CL`)
3. Create a new **Data Collection Rule**, e.g. `dcr-webapp`, in the **same region** as the workspace
4. Upload a sample of `webapp.log` and set the record delimiter to **end of line**
5. Collection path: `/var/log/webapp/webapp.log`
6. Keep the default columns `TimeGenerated` and `RawData` (fields are parsed later in the query)
7. Under **Resources**, add your VM. Azure Monitor Agent is installed automatically.
8. **Review + create**

---

## Step 4: Verify data is arriving

Wait 5 to 10 minutes, then open the workspace -> **Logs** and run:

```kusto
WebAppLogs_CL
| take 10
```

If it returns rows, ingestion works.

---

## Step 5: Parse the logs and save a function

```kusto
WebAppLogs_CL
| extend Level   = extract(@"^\S+\s+(\w+)\s", 1, RawData),
         Message = extract(@"^\S+\s+\w+\s+(.*)$", 1, RawData)
```

Click **Save** -> **Save as function**, name it `WebAppLogs`. The queries below use this function.

---

## Step 6: Dashboard queries

### Logs over time by level (time chart)

```kusto
WebAppLogs
| summarize Count = count() by bin(TimeGenerated, 1m), Level
| render timechart
```

### Level distribution (pie chart)

```kusto
WebAppLogs
| summarize Count = count() by Level
| render piechart
```

### Errors in the last hour (single number)

```kusto
WebAppLogs
| where TimeGenerated > ago(1h) and Level == "ERROR"
| summarize ErrorCount = count()
```

### Top error types (bar chart)

```kusto
WebAppLogs
| where Level == "ERROR"
| extend Pattern = replace_regex(Message, @"\d+", "N")
| summarize Count = count() by Pattern
| top 5 by Count desc
| render barchart
```

### Average API response time (time chart)

```kusto
WebAppLogs
| where Message startswith "API response time"
| extend Ms = toint(extract(@"(\d+)ms", 1, Message))
| summarize AvgMs = avg(Ms) by bin(TimeGenerated, 5m)
| render timechart
```

### Latest errors and warnings (table)

```kusto
WebAppLogs
| where Level in ("ERROR", "WARNING")
| project TimeGenerated, Level, Message
| order by TimeGenerated desc
| take 20
```

---

## Step 7: Build the dashboard

### Option A: Azure Dashboard (quick)

1. Run a query in **Logs**
2. Click **Pin to** -> **Azure dashboard** -> **Create new** -> name it `WebApp Logs`
3. Repeat for each query (choose the same dashboard)
4. Portal -> **Dashboard** -> open `WebApp Logs` -> **Edit** to resize and arrange tiles
5. **Share** the dashboard if others need access

### Option B: Azure Workbook (recommended, more flexible)

1. Workspace -> **Workbooks** -> **New**
2. **Add** -> **Add parameter**: a time range picker (and optionally a Level dropdown)
3. **Add** -> **Add query**: paste a query, pick the visualization (time chart, pie, grid, tiles)
4. Repeat for each panel, then **Save** to the resource group

### Option C: Azure Managed Grafana

Create an Azure Managed Grafana instance, add the Log Analytics workspace as an Azure Monitor data source, and reuse the same KQL in panels.

---

## Step 8 (optional): Alert on errors

1. Run the "Errors in the last hour" query in **Logs**
2. **New alert rule** -> condition: count greater than your threshold (e.g. 20)
3. Add an **action group** (email, SMS or webhook)

---

## Troubleshooting

| Problem | What to check |
|---------|---------------|
| No rows in `WebAppLogs_CL` | Wait 10 minutes; confirm DCR is attached to the VM; check file path |
| Agent not running | `sudo systemctl status azuremonitoragent` |
| Permission errors | File must be readable: `sudo chmod 644 /var/log/webapp/webapp.log` |
| DCR and workspace region mismatch | Recreate the DCR in the same region as the workspace |
| `Level` or `Message` is empty | Check the log format matches the regex in Step 5 |
| Timestamps all look like ingestion time | Expected: `TimeGenerated` is ingestion time unless parsed in a transformation |
| Cron job does nothing | `grep CRON /var/log/syslog` and `cat /tmp/generate_logs.cron.log` for errors |
| `python3: not found` in cron | Use the full path from `which python3` |
| Permission denied in cron | Use `sudo crontab -e`, or write to your own folder with `-f` |
| Runs pile up | Use `-n` and `flock -n`; make sure the script is the improved version |
| Unexpected cost | Stop the generator: `sudo pkill -f generate_dummy_logs.py` and remove the cron job |

---

## Cleanup

```bash
# Stop the generator on the VM
sudo pkill -f generate_dummy_logs.py

# Remove the cron job (delete the line, or remove all root cron jobs)
sudo crontab -e
sudo crontab -r
```

Delete the resource group (`rg-logs`) in the portal to remove the workspace, DCR and dashboard resources.
