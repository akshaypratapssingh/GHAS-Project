---
description: >
  Workflow 1 Coordinator — delegates fetch → sort → Jira to sub-agents in sequence.
  Does NOT execute any steps directly. Passes output from each sub-agent to the next.
tools: []
---

# Alert Ingestor — Workflow 1 Coordinator

You are the **coordinator** for Workflow 1.
Your only job is to invoke sub-agents in the correct order and pass data between them.
You do NOT call any tools directly — each sub-agent owns its own tools.

---

## Agent Chain

```
@alert-ingestor
      │
      ├── Step 1 ──▶ @w1-fetcher
      │                └── runs fetch script → produces Excel
      │                └── returns: excel_path, alert_counts
      │
      ├── Step 2 ──▶ @w1-sorter
      │                └── sorts Excel by service + severity
      │                └── returns: sorted excel_path, grouped_alerts
      │
      └── Step 3 ──▶ @w1-jira-manager
                       └── dedup check + create/update Jira tickets
                       └── returns: jira_results, updated excel_path
```

---

## Input (from @dependabot-vuln-orchestrator)

| Field | Example | Required |
|---|---|---|
| `PROJECT_KEY` | `SCRUM` | ✅ (default: read from `JIRA_PROJECT_KEY` in `.env`) |
| `REPO_ROOT` | `C:/Users/AkshayPratapSingh/Downloads/GHAS-Project/GHAS-Project` | ✅ (default: read from `REPO_ROOT` in `.env`) |

---

## Step 1 — Invoke @w1-fetcher

Call `@w1-fetcher` with:
```
REPO_ROOT = <REPO_ROOT>
```

Wait for it to complete.

**On success** — receive:
- `excel_path` — full path to `dependabot_alerts_<timestamp>.xlsx`
- `total_alerts` — total count
- `alert_counts` — `{ CRITICAL: X, HIGH: X, MEDIUM: X, LOW: X }`

**On failure** — stop the entire workflow. Report to `@dependabot-vuln-orchestrator`:
```
W1 FAILED at @w1-fetcher
Reason: <exact error from @w1-fetcher>
```

---

## Step 2 — Invoke @w1-sorter

Call `@w1-sorter` with output from Step 1:
```
excel_path   = <from @w1-fetcher>
total_alerts = <from @w1-fetcher>
alert_counts = <from @w1-fetcher>
```

Wait for it to complete.

**On success** — receive:
- `excel_path` — updated sorted Excel path
- `grouped_alerts` — `{ "GHS": [{alert1}, ...], "service-2": [...] }`
- `service_names` — list of unique service names found

**On failure** — stop. Report to `@dependabot-vuln-orchestrator`:
```
W1 FAILED at @w1-sorter
Reason: <exact error from @w1-sorter>
```

---

## Step 3 — Invoke @w1-jira-manager

Call `@w1-jira-manager` with output from Step 2:
```
excel_path      = <from @w1-sorter>
grouped_alerts  = <from @w1-sorter>
service_names   = <from @w1-sorter>
PROJECT_KEY     = <from orchestrator input>
```

Wait for it to complete.

**On success** — receive:
- `jira_results` — `{ created: [{service, key}], skipped: [...], failed: [...] }`
- `excel_path` — final Excel with Jira Key + Status columns filled

**On failure (partial)** — @w1-jira-manager handles per-service failures internally.
Collect whatever results it returns and proceed to the final summary.

---

## Final Summary

Report back to `@dependabot-vuln-orchestrator`:

```
╔══════════════════════════════════════════════════════════════╗
║         WORKFLOW 1 — ALERT INGESTOR COMPLETE                 ║
╠══════════════════════════════════════════════════════════════╣
║  Agent chain      : @w1-fetcher → @w1-sorter                 ║
║                     → @w1-jira-manager                       ║
║  Excel report     : dependabot_alerts_<timestamp>.xlsx        ║
║  Services found   : X                                         ║
║  Total alerts     : X  (C-X | H-X | M-X | L-X)               ║
╠══════════════════════════════════════════════════════════════╣
║  Jira CREATED     : X  → [SCRUM-5, SCRUM-6, ...]             ║
║  Jira SKIPPED     : X  → [SCRUM-3, ...]  (already exist)     ║
║  Jira FAILED      : X  → (see errors)                        ║
╠══════════════════════════════════════════════════════════════╣
║  Services ready for Workflow 2:                               ║
║    • GHS → SCRUM-5                                            ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Rules

1. **Never call tools directly** — delegate all work to `@w1-fetcher`, `@w1-sorter`, `@w1-jira-manager`
2. **Always invoke in order** — Step 1 → Step 2 → Step 3. Never skip or reorder.
3. **Always pass full output** of each sub-agent to the next — no data loss between steps
4. **Stop on fetcher failure** — no point sorting or creating Jira tickets if Excel has no data
5. **Never stop on Jira failures** — @w1-jira-manager handles partial failures; collect what it returns

---

## Input

| Field | Example | Required |
|-------|---------|----------|
| `PROJECT_KEY` | `SCRUM` | ✅ (default: read from `JIRA_PROJECT_KEY` in `.env`) |
| `REPO_ROOT` | `C:/Users/AkshayPratapSingh/Downloads/GHAS-Project/GHAS-Project` | ✅ (default: read from `REPO_ROOT` in `.env`) |

---

## ═══════════════════════════════════════════
## STEP 1 — FETCH ALERTS (run the script)
## ═══════════════════════════════════════════

### 1.1 — Install dependencies
```bash
pip install requests openpyxl python-dotenv
```

### 1.2 — Run the fetch script
```bash
cd <REPO_ROOT>
python .github/scripts/fetch_dependabot_alerts.py
```

The script reads `GITHUB_TOKEN` from the `.env` file at the repo root.

### 1.3 — Verify output

- Confirm a file named `dependabot_alerts_<timestamp>.xlsx` was created in the current directory
- Confirm it has at least one data row (not just the header)

**If `GITHUB_TOKEN` is not set:**
> Stop. Tell the user: "Set GITHUB_TOKEN in a .env file at the repo root and re-run."

**If the script throws an error:**
> Stop. Return the exact error message to the user.

**If the file is empty (header only):**
> Stop. Print: `✅ No open Maven Dependabot alerts found. Nothing to do.`

Print confirmation:
```
[FETCH] ✅ Excel created: dependabot_alerts_<timestamp>.xlsx
         Total alerts : X  (CRITICAL: X | HIGH: X | MEDIUM: X | LOW: X)
```

---

## ═══════════════════════════════════════════
## STEP 2 — SORT THE EXCEL
## ═══════════════════════════════════════════

Open the Excel file and sort the "Alerts" sheet:

- **Primary sort:** Service Name (column A) — alphabetical A → Z
- **Secondary sort:** Severity (column D) — CRITICAL → HIGH → MEDIUM → LOW

Severity sort order:
```
CRITICAL = 0 | HIGH = 1 | MEDIUM = 2 | LOW = 3
```

After sorting, **update the "Summary" sheet** with per-service counts:

| Service | CRITICAL | HIGH | MEDIUM | LOW | Total |
|---------|----------|------|--------|-----|-------|

Save and close the file.

Then build a grouped structure in memory:
```
{
  "HMS":       [ {alert1}, {alert2}, ... ],   ← CRITICAL first within each service
  "service-2": [ {alert1}, ... ],
}
```

Print confirmation:
```
[SORT] ✅ Excel sorted and grouped.
        Services found: HMS, service-2, ...
```

---

## ═══════════════════════════════════════════
## STEP 3 — DEDUP CHECK IN JIRA
## ═══════════════════════════════════════════

Run the Jira manager script (handles dedup + create/update automatically):

```bash
cd <REPO_ROOT>
python .github/scripts/create_jira_tickets.py <excel_path>
```

The script reads `JIRA_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`, `JIRA_PROJECT_KEY` from the `.env` file.

For **each service** in the grouped structure, it runs this JQL search internally:

```jql
project = "<PROJECT_KEY>"
AND labels = "<SERVICE_NAME>"
AND labels = "dependabot"
AND statusCategory in ("To Do", "In Progress")
```

Record the result per service:
- **Ticket found** → mark as `UPDATE` — recalculate counts, update title + description + add comment
- **No ticket found** → mark as `CREATE`

Print per-service result:
```
[JIRA] Processing: GHAS-Project  (C-3 H-6 M-5 L-1)
  → No existing ticket found — will CREATE
  ✅ Created: SCRUM-5

[JIRA] Processing: service-2     (C-1 H-2 M-0 L-0)
  → Found existing ticket: SCRUM-3 — will UPDATE
  ✅ Updated: SCRUM-3
```

**If Jira search fails for a service:**
> Log the error, mark as `FAILED`, continue with the next service. Do not stop.

---

## ═══════════════════════════════════════════
## STEP 4 — JIRA TICKET FIELDS
## ═══════════════════════════════════════════

The `create_jira_tickets.py` script creates each ticket with:

### Ticket Title format:
```
<SERVICE_NAME> (C-<n>, H-<n>, M-<n>, L-<n>)
```
Only includes severity labels that have at least 1 alert:
- `GHAS-Project (C-3, H-6, M-5, L-1)`

---

### Field values:

| Field | Value |
|-------|-------|
| Project | `SCRUM` |
| Issue Type | Bug |
| Summary | `<SERVICE_NAME> (C-<n>, H-<n>, M-<n>, L-<n>)` |
| Priority | CRITICAL→Highest · HIGH→High · MEDIUM→Medium · LOW→Low |
| Labels | `GHAS`, `<SERVICE_NAME>`, `dependabot`, `security` |
| Description | ADF table grouped by severity (GHSA ID, CVE, Package, Vulnerable Range, Safe Version, Summary) |

---

### Step 4.1 — Excel update (automatic)

The script updates **every row** of each service in the "Alerts" sheet:
- **Column N (Jira Key):** e.g. `SCRUM-5`
- **Column O (Jira Status):** `CREATED`, `UPDATED`, or `FAILED`

Excel is saved **once after all services** are processed.

---

## ═══════════════════════════════════════════
## FINAL SUMMARY
## ═══════════════════════════════════════════

```
╔══════════════════════════════════════════════════════════════╗
║         WORKFLOW 1 — ALERT INGESTOR COMPLETE                 ║
╠══════════════════════════════════════════════════════════════╣
║  Excel report     : dependabot_alerts_<timestamp>.xlsx        ║
║  Services found   : X                                         ║
║  Total alerts     : X  (C-X | H-X | M-X | L-X)               ║
╠══════════════════════════════════════════════════════════════╣
║  Jira CREATED     : X  → [SEC-101, SEC-102, ...]              ║
║  Jira SKIPPED     : X  → [SEC-098, ...]  (already exist)      ║
║  Jira FAILED      : X  → (see errors below)                   ║
╠══════════════════════════════════════════════════════════════╣
║  Services with new tickets (ready for Workflow 2):            ║
║    • HMS       → SEC-101                                      ║
║    • service-2 → SEC-102                                      ║
╚══════════════════════════════════════════════════════════════╝
```

Pass the services + Jira keys to `@dependabot-vuln-orchestrator` if running in "both" mode.

---

## Hard Rules

1. **One ticket per service** — never one ticket per CVE
2. **Always check Jira before creating** — never create duplicates
3. **Skipped = existing open ticket found** — do not create a second one
4. **If Jira search fails** → log the error, skip that service, continue — do not stop entirely
5. **If ticket creation fails** → log the failure, continue with remaining services
6. **Always save Excel after ALL services** are processed — not after each one
7. **Never stop the whole run** because one service's Jira call failed
