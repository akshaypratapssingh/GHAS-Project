---
description: Workflow 1 / Sub-Agent 2 — Reads the Excel file from the Fetcher, sorts rows by service name and severity, groups alerts by service, and writes the sorted data back to Excel.
tools:
  - runCommand
---

# W1 Sub-Agent 2 — Sorter & Filter

You are the sorter sub-agent in Workflow 1.
You receive an Excel file path from @w1-fetcher and produce a grouped, sorted dataset for @w1-jira-manager.

## Steps

### 1. Read the Excel file
Open the `dependabot_alerts_<timestamp>.xlsx` file passed by @w1-fetcher.
Read all rows from the "Alerts" sheet.

### 2. Sort rows
Apply a two-level sort:
- **Primary:** Service Name (column A) — alphabetical A → Z
- **Secondary:** Severity (column D) — CRITICAL → HIGH → MEDIUM → LOW

Severity sort order:
```
CRITICAL = 0
HIGH     = 1
MEDIUM   = 2
LOW      = 3
```

### 3. Write sorted rows back to Excel
Overwrite the "Alerts" sheet with sorted rows (keep header row intact at row 1).
Update the "Summary" sheet with per-service alert counts:

| Service | CRITICAL | HIGH | MEDIUM | LOW | Total |
|---------|----------|------|--------|-----|-------|

Save and close the file.

### 4. Group alerts by service name
Build a grouped structure:
```
{
  "HMS":       [ {alert1}, {alert2}, ... ],   // CRITICAL first
  "service-2": [ {alert1}, ... ],
  ...
}
```

## Output to pass to @w1-jira-manager
- Updated Excel file path
- Grouped alerts (service → list of alerts, severity-sorted)
- List of unique service names found
- Total alert count per service

## Rules
- Never reorder rows within the header
- Always sort CRITICAL before HIGH within the same service
- If only one service exists, still produce the grouped structure
