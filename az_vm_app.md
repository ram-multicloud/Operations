# Azure VM Monitoring Guide: Logs, KQL and Dashboard

**What you will build**

```text
gen_logs.py  →  /var/log/webapp/webapp.log  →  Azure Monitor Agent  →  Log Analytics (WebApp_CL)  →  KQL queries  →  Dashboard
   (VM)               (JSON lines)                (DCR on the VM)           (table)                    (charts)
```

Contents

1. Create `gen_logs.py`
2. Run it
3. Set up Azure monitoring
4. What is KQL and how it works
5. KQL queries for the sample logs
6. Create the monitoring dashboard
7. Troubleshooting

---

## 1. Create `gen_logs.py`

The script writes one JSON object per line. The keys match the Azure table columns exactly, including `TimeGenerated`, which Azure requires.

On the VM:

```bash
nano ~/gen_logs.py
```

Paste this, then save with `Ctrl+O`, Enter, `Ctrl+X`:

```python
#!/usr/bin/env python3
"""gen_logs.py - generate sample web-app logs as JSON Lines."""
import argparse
import json
import random
import time
from datetime import datetime, timezone

PATHS = ["/api/orders", "/api/users", "/api/cart", "/login", "/health"]
METHODS = ["GET", "POST"]


def make_record():
    status = random.choices([200, 201, 404, 500], [80, 8, 7, 5])[0]
    return {
        "TimeGenerated": datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"),
        "Level": "ERROR" if status >= 500 else "WARN" if status >= 400 else "INFO",
        "Method": random.choice(METHODS),
        "Path": random.choice(PATHS),
        "Status": status,
        "LatencyMs": random.randint(20, 900),
    }


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--output", default="/var/log/webapp/webapp.log")
    ap.add_argument("--count", type=int, default=0, help="0 = run forever")
    ap.add_argument("--stdout", action="store_true")
    args = ap.parse_args()

    written = 0
    try:
        with open(args.output, "a", buffering=1) as f:
            while args.count == 0 or written < args.count:
                line = json.dumps(make_record())      # one object per line
                f.write(line + "\n")
                if args.stdout:
                    print(line)
                written += 1
                time.sleep(random.uniform(0.2, 2))
    except PermissionError:
        raise SystemExit(f"Cannot write to {args.output}. Create the folder and fix permissions first.")
    except KeyboardInterrupt:
        pass


if __name__ == "__main__":
    main()
```

Sample output line:

```json
{"TimeGenerated": "2026-11-13T10:15:02Z", "Level": "INFO", "Method": "GET", "Path": "/api/orders", "Status": 200, "LatencyMs": 123}
```

---

## 2. Run It

```bash
# 1. Check Python (use python3, not python)
python3 --version

# 2. Create the log folder (first time only)
sudo mkdir -p /var/log/webapp
sudo chmod 777 /var/log/webapp

# 3. Move any old plain-text log aside so it doesn't mix with JSON
[ -f /var/log/webapp/webapp.log ] && sudo mv /var/log/webapp/webapp.log /var/log/webapp/webapp.old

# 4. Quick test: write 5 lines and print them
python3 ~/gen_logs.py --count 5 --stdout

# 5. Run in the background
nohup python3 ~/gen_logs.py > /dev/null 2>&1 &

# 6. Verify
tail -n 5 /var/log/webapp/webapp.log
pgrep -af gen_logs.py
```

Stop it with: `pkill -f gen_logs.py`

---

## 3. Set Up Azure Monitoring

> Portal menu names change from time to time, so some labels may differ slightly.

### Step 1: Create a Log Analytics workspace

1. Azure portal → **Log Analytics workspaces** → **Create**.
2. Pick the subscription and resource group, name it `law-webapp`, and choose the **same region as the VM**.
3. **Review + create**.

### Step 2: Create the custom table

1. Open the workspace → **Tables** → **Create** → the custom log option based on the Azure Monitor Agent (AMA).
2. Table name: `WebApp_CL` (custom tables end in `_CL`).
3. Upload a sample file that includes `TimeGenerated` (for example `sample_webapp.log` or `sample_webapp.json`).
4. Make sure the table has exactly these columns:

| Column name | Type in the table editor | Type in the JSON schema | Example value |
|---|---|---|---|
| `TimeGenerated` | DateTime | `datetime` | `2026-11-13T10:15:02Z` |
| `Level` | String | `string` | `INFO`, `WARN`, `ERROR` |
| `Method` | String | `string` | `GET`, `POST` |
| `Path` | String | `string` | `/api/orders` |
| `Status` | Int | `int` | `200` |
| `LatencyMs` | Int | `int` | `123` |

5. In the **Transformation editor**, if the preview shows all six columns, keep `source` and click **Next**, then **Create**.

Notes:

- Column names are case-sensitive and must match the keys in your log lines exactly (`Status`, not `status`).
- Pick **Int** for `Status` and `LatencyMs`, not Long or Real, so the types match the JSON schema.
- The error "TimeGenerated field is not found in the sample" means the sample has no `TimeGenerated`. Upload a sample that includes it, as the generator's output does.

### Step 3: Create the data collection rule (DCR)

1. **Monitor** → **Data Collection Rules** → **Create**.
2. Name `dcr-webapp`, same region as the workspace, Platform **Linux**.
3. **Resources** → add your VM (this installs the Azure Monitor Agent automatically).
4. **Collect and deliver** → **Add data source** and fill in the form:

| Field | Value |
|---|---|
| Data source type | `Custom JSON Logs` |
| File pattern | `/var/log/webapp/webapp.log` |
| Table name | `WebApp_CL` |
| Transform | `source` |
| JSON Schema | the column list below |

```json
[
  {"name": "TimeGenerated", "type": "datetime"},
  {"name": "Level",         "type": "string"},
  {"name": "Method",        "type": "string"},
  {"name": "Path",          "type": "string"},
  {"name": "Status",        "type": "int"},
  {"name": "LatencyMs",     "type": "int"}
]
```

5. Destination: **Azure Monitor Logs** → your workspace. A custom JSON log data source can have only one workspace per DCR.
6. Add a second data source, **Performance Counters** (Basic), with the same workspace, for CPU and memory.
7. **Review + create**.

Notes:

- The file pattern is the path on the VM's local disk. Wildcards work, for example `/var/log/webapp/*.log`.
- `TimeGenerated` is the only required schema property. If it is missing, the ingestion time is used instead.
- Schema types must match the table column types (`Status` and `LatencyMs` are `int` in both places).
- If the portal pre-fills the JSON Schema box with a sample, keep its format and change the names and types to match the table above.
- Use **Custom JSON Logs**, not Custom Text Logs. With JSON logs each field becomes its own column, so no `parse_json` transform is needed.

Shortcut for built-in VM charts: VM → **Monitoring** → **Insights** → **Enable**.

### Step 4: Verify

```bash
sudo systemctl status azuremonitoragent
```

It should show `active (running)`. Wait 5 to 10 minutes, then in the workspace open **Logs** and run:

```kql
WebApp_CL
| take 10
```

---

## 4. What Is KQL and How Does It Work?

**KQL (Kusto Query Language)** is the read-only query language used by Azure Monitor / Log Analytics, Microsoft Sentinel, Azure Data Explorer and Application Insights. You use it to search, filter, aggregate and chart your logs.

### 4.1 The core idea: a pipeline

A KQL query starts with a **table** and passes the rows through a series of operators joined by the pipe `|`. Each operator takes the table from the step before, transforms it, and passes it on. This is like a Unix pipeline.

```kql
WebApp_CL                                   // 1. start with all rows
| where TimeGenerated > ago(1h)             // 2. keep only the last hour
| where Status >= 500                       // 3. keep only server errors
| summarize Errors = count() by Path        // 4. count them per path
| order by Errors desc                      // 5. sort, biggest first
```

Each step only sees the output of the previous step. So **filter early** (`where` first) to make queries fast and cheap.

### 4.2 Execution flow

```text
Table (all rows)
   │  where     → fewer rows
   │  extend    → adds computed columns
   │  summarize → groups rows into aggregates
   │  project   → chooses which columns to keep
   │  order by / top → sorts / limits
   ▼
Result table → shown as grid or chart (render)
```

### 4.3 Operator cheat sheet

| Operator | What it does | Example |
|---|---|---|
| `where` | Filter rows | `where Status >= 500` |
| `project` | Choose or rename columns | `project TimeGenerated, Path, Status` |
| `project-away` | Remove columns | `project-away RawData` |
| `extend` | Add a computed column | `extend IsError = Status >= 500` |
| `summarize` | Aggregate (like SQL GROUP BY) | `summarize count() by Path` |
| `bin()` | Round time into buckets | `by bin(TimeGenerated, 5m)` |
| `order by` / `sort by` | Sort | `order by Requests desc` |
| `top N by` | Top N rows | `top 5 by LatencyMs desc` |
| `take N` | Any N rows (quick peek) | `take 10` |
| `count` | Count rows | `WebApp_CL \| count` |
| `distinct` | Unique values | `distinct Path` |
| `render` | Draw a chart | `render timechart` |
| `parse_json()` | Read JSON text | `parse_json(RawData)` |
| `ago()` | Relative time | `ago(30m)`, `ago(1d)` |
| `between` | Range filter | `where Status between (400 .. 499)` |
| `join` / `union` | Combine tables | `union WebApp_CL, Perf` |

### 4.4 Aggregate functions (used inside `summarize`)

`count()`, `countif(condition)`, `sum(x)`, `avg(x)`, `min(x)`, `max(x)`, `percentile(x, 95)`, `dcount(x)` (distinct count), `make_list(x)`.

### 4.5 Useful details

- **Case:** table and column names are case-sensitive (`Status` ≠ `status`). `==` is case-sensitive; `=~` ignores case.
- **Strings:** use `contains`, `startswith`, `has` (fastest for whole terms), `matches regex`.
- **Time units:** `s` seconds, `m` minutes, `h` hours, `d` days (e.g. `5m`, `1h`).
- **Comments:** `//` to end of line.
- **Time range:** the Logs blade has a time picker. A `where TimeGenerated > ago(...)` in the query overrides or narrows it.
- **Pinned charts** keep the query and `render` line, and refresh when the dashboard loads.
- **Types:** use `toint()`, `todouble()`, `tostring()`, `todatetime()` to convert.

---

## 5. KQL Queries for the Sample Logs

The table is `WebApp_CL`, with columns `TimeGenerated`, `Level`, `Method`, `Path`, `Status`, `LatencyMs`.

> With Custom JSON Logs every field is already its own column, so these queries need no `parse_json`.

### 5.1 Explore

**Latest 20 log lines**
```kql
WebApp_CL
| order by TimeGenerated desc
| take 20
```

**Total number of logs in the last hour**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| count
```

**Only errors, newest first**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| where Level == "ERROR"
| project TimeGenerated, Method, Path, Status, LatencyMs
| order by TimeGenerated desc
```

### 5.2 Dashboard tiles

**Tile 1: Requests per minute by status code (line chart)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| summarize Requests = count() by bin(TimeGenerated, 1m), Status = tostring(Status)
| render timechart
```

**Tile 2: Requests by log level (pie chart)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| summarize Count = count() by Level
| render piechart
```

**Tile 3: Top 5 busiest paths (bar chart)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| summarize Requests = count() by Path
| top 5 by Requests desc
| render barchart
```

**Tile 4: Server errors (5xx) by path (bar chart)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h) and Status >= 500
| summarize Errors = count() by Path
| order by Errors desc
| render barchart
```

**Tile 5: Error rate % over time (line chart)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| summarize Total = count(), Errors = countif(Status >= 500) by bin(TimeGenerated, 5m)
| extend ErrorRatePct = round(100.0 * Errors / Total, 2)
| project TimeGenerated, ErrorRatePct
| render timechart
```

**Tile 6: Average and 95th percentile latency (line chart)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| summarize AvgMs = avg(LatencyMs), P95Ms = percentile(LatencyMs, 95) by bin(TimeGenerated, 5m)
| render timechart
```

**Tile 7: Slowest 10 requests (table)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| top 10 by LatencyMs desc
| project TimeGenerated, Method, Path, Status, LatencyMs
```

**Tile 8: Summary numbers (table with one row)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h)
| summarize TotalRequests = count(),
            Errors5xx = countif(Status >= 500),
            Warnings4xx = countif(Status between (400 .. 499)),
            AvgLatencyMs = round(avg(LatencyMs), 1)
```

**Tile 9: 4xx vs 5xx over time (stacked bars)**
```kql
WebApp_CL
| where TimeGenerated > ago(1h) and Status >= 400
| extend Class = iff(Status >= 500, "5xx server error", "4xx client error")
| summarize Count = count() by bin(TimeGenerated, 5m), Class
| render columnchart
```

### 5.3 VM health (from the Performance Counters data source)

**CPU %**
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize CPU = avg(CounterValue) by bin(TimeGenerated, 5m)
| render timechart
```

**Memory used %**
```kql
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Memory" and CounterName == "% Used Memory"
| summarize MemoryPct = avg(CounterValue) by bin(TimeGenerated, 5m)
| render timechart
```

### 5.4 Alert idea

Use Tile 4's query, then **New alert rule** → condition: more than 10 rows in the last 5 minutes → action group (email).

---

## 6. Create the Monitoring Dashboard

### Method A: Azure Dashboard (pin tiles)

1. Open the Log Analytics workspace → **Logs**.
2. Paste Tile 1's query → **Run**.
3. Click **Pin to** → **Azure dashboard** → **Create new** → name it `WebApp Monitoring` → **Pin**.
4. Repeat for each tile query (choose **Existing** and select `WebApp Monitoring` from the second tile onward).
5. Open **Dashboard** from the portal menu → select `WebApp Monitoring`.
6. Click **Edit**, drag and resize the tiles, then **Save**.
7. Set auto-refresh in the dashboard toolbar (for example every 5 minutes) and use **Share** if others need access.

Suggested layout:

```text
┌───────────────────────┬──────────────────────┐
│ Tile 8: Summary       │ Tile 5: Error rate % │
├───────────────────────┼──────────────────────┤
│ Tile 1: Requests/min  │ Tile 6: Latency      │
├──────────┬────────────┼──────────────────────┤
│ Tile 2   │ Tile 3     │ Tile 4: 5xx by path  │
├──────────┴────────────┼──────────────────────┤
│ Tile 7: Slowest       │ CPU and Memory       │
└───────────────────────┴──────────────────────┘
```

### Method B: Azure Workbook (interactive, recommended for more features)

1. **Monitor** → **Workbooks** → **New**.
2. **Add** → **Add query** → pick your workspace → paste a query → choose visualization → **Done editing**.
3. Add a **Parameters** block for the time range so viewers can switch between 15 minutes, 1 hour and 1 day. Then use `{TimeRange}` in the queries.
4. **Save** and pin it to a dashboard if you want.

### Method C: Grafana

**Azure Managed Grafana** can use the Azure Monitor data source to query the same workspace with the same KQL.

---

## 7. Troubleshooting

| Problem | Check |
|---|---|
| `python: command not found` | Use `python3`. |
| `Cannot write to /var/log/webapp/webapp.log` | Run the folder commands in section 2. |
| "TimeGenerated not found" in the wizard | Upload a sample that includes `TimeGenerated` (the generator writes it), or add it in the transform. |
| Table has no rows | Is the agent `active (running)`? Is the file path exact? Is the generator still running? Wait 10 minutes. |
| Columns are empty or null | The JSON schema names and types must match the table columns and the log keys exactly (case-sensitive). Use `int`, not long or real, for `Status` and `LatencyMs`. |
| Query returns nothing | Widen the time range, since `ago(1h)` only shows the last hour. |
| `Perf` is empty | Add the Performance Counters data source to the DCR with the workspace as destination. |
| Permission errors in Azure | You need at least Contributor on the workspace and resource group. |

**Cost note:** Log Analytics bills by data ingested. This sample is tiny, but stop the generator and delete the DCR and workspace when you finish testing:

```bash
pkill -f gen_logs.py
```

## Reference Links

- Collect logs from a text file with Azure Monitor Agent: https://learn.microsoft.com/azure/azure-monitor/agents/data-collection-text-log
- KQL overview: https://learn.microsoft.com/azure/data-explorer/kusto/query/
- Azure Workbooks: https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-overview
