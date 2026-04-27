---
name: sf-pipeline-dashboard
description: Pull Salesforce opportunities and call reports matching user-specified keywords, then generate an interactive HTML dashboard with notes and SF links. Use when asked to build a pipeline dashboard or search Salesforce for activity around specific topics.
---

# Salesforce Pipeline Dashboard Generator

You are generating an interactive HTML dashboard from live Salesforce data.

## Inputs (from `$ARGUMENTS`)

Parse the following from the user's input:
- **keywords** — one or more terms to search for (e.g. "CrossX, ECN, Crossover")
- **timeframe** — number of days to look back (default: `365` if not specified)
- **output filename** — default: `dashboard-YYYY-MM-DD.html` using today's date

If keywords are missing, ask before proceeding. Timeframe and filename can always default.

---

## Step 1 — Resolve Salesforce org

Use `get_username` (defaultTargetOrg: true) from the `salesforce-dx` MCP to get the org username.
Directory: use the current working directory.

Store as `SF_USER` for all subsequent queries.

---

## Step 2 — Get instance URL

Run this shell command to get the Lightning base URL:
```bash
sf org display --target-org <SF_USER> --json 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('result',{}).get('instanceUrl','https://bitgo.lightning.force.com'))"
```

Store as `SF_BASE` (e.g. `https://bitgo.lightning.force.com`).

---

## Step 3 — Query Opportunities

Build a SOQL query filtering on `Name` only (Description is not filterable).
Use `LAST_N_DAYS:<timeframe>` for the date filter on `CreatedDate`.

```sql
SELECT Id, Name, StageName, Amount, CloseDate, Account.Name, Owner.Name, CreatedDate
FROM Opportunity
WHERE CreatedDate >= LAST_N_DAYS:<timeframe>
AND (<Name LIKE clauses for each keyword, joined with OR>)
ORDER BY CreatedDate DESC
```

Example LIKE clause for keywords ["CrossX", "ECN", "Crossover"]:
```
Name LIKE '%CrossX%' OR Name LIKE '%ECN%' OR Name LIKE '%Crossover%'
```

Run via `run_soql_query`. Collect all returned records.

---

## Step 4 — Query Email / Activity Threads (Tasks)

Build a SOQL query on `Subject` only (Description is not filterable on Task either).

```sql
SELECT Id, Subject, Description, ActivityDate, Type, Status, Owner.Name, Account.Name
FROM Task
WHERE ActivityDate >= LAST_N_DAYS:<timeframe>
AND (<Subject LIKE clauses for each keyword, joined with OR>)
ORDER BY ActivityDate DESC
```

**Important:** Task results can be very large. If the result is saved to a file rather than returned inline, read it back using Python:

```python
import json
with open('<result_file_path>') as f:
    data = json.load(f)
text = data[0]['text']
json_start = text.index('{')
result = json.loads(text[json_start:])
records = result['records']
```

Then filter records where Subject (case-insensitive) contains any keyword:
```python
terms = [k.lower() for k in keywords]
matches = [r for r in records if any(t in (r.get('Subject') or '').lower() for t in terms)]
```

Deduplicate email threads by normalizing subjects (strip "Re:", "RE:", "Fwd:", "FW:" prefixes) and grouping by (Account, normalized_subject). Keep the most recent record per group.

Separate **calls/meetings** (Type in ['Call','Meeting','Demo','In Person Meeting'] or Subject not starting with 'Email:') from email threads. Present calls first.

---

## Step 5 — Pull Notes for Opportunities

For each opportunity returned in Step 3, query related Tasks to get actual call/meeting notes.

```sql
SELECT Id, Subject, Description, ActivityDate, Type, Owner.Name, WhatId
FROM Task
WHERE WhatId IN ('<opp_id_1>','<opp_id_2>',...)
AND ActivityDate >= LAST_N_DAYS:<timeframe>
AND Subject NOT LIKE 'Email: Sales Won%'
AND Subject NOT LIKE 'Email: Case%'
ORDER BY WhatId, ActivityDate DESC
```

**Do not include** automated system emails (Subject LIKE 'Email: Sales Won%' or 'Email: Case Received%') — these contain no useful notes.

Group results by `WhatId` to build a notes map: `{ oppId: [ {date, type, owner, text}, ... ] }`.

---

## Step 6 — Determine keyword match per record

For each opportunity and activity, tag which keywords matched:
- Check `Name` (opps) or `Subject` (tasks) case-insensitively against each keyword
- Store as an array on the record: `kw: ["crossx", "ecn"]`
- Use lowercase slugs for keyword keys; display names should match what the user entered

---

## Step 7 — Generate the HTML dashboard

Write a single self-contained HTML file to `salesforce-dashboard/<output_filename>` (or the current directory if not in salesforce-dashboard).

The dashboard must include:

### Summary stat cards (top)
- Total Opportunities matched
- Closed Won count
- In Progress count (non-closed stages)
- Closed Lost count
- Email/Activity thread count

### Filter controls
- **Keyword chips** — one per keyword, all active by default; clicking toggles that keyword on/off
- **Search box** — filters account name + opportunity/subject text live
- **Owner dropdown** — populated from unique owners in the data
- **Stage dropdown** — Opportunities tab only

### Two tabs
1. **Opportunities** — columns: Date, Account, Opportunity Name, Stage (color-coded badge), Owner, Keywords, Notes, SF Link
2. **Email Activity** — columns: Date, Owner, Account, Subject, Keywords

### Notes column behavior
- If no notes: show `—`
- If notes exist: show a 1-line preview of the first note + a "View note / View N notes" button
- Button opens a modal showing each note in a separate block with: activity type, date, owner, full text (pre-wrap)
- Modal closes on × button, backdrop click, or Escape key

### SF Link column
- Each opportunity row gets an "↗ Open" link to `<SF_BASE>/<oppId>` opening in a new tab

### All filtering/sorting is client-side JavaScript — no server required.

### Color scheme for stages
- Closed Won: green
- Closed Lost: red
- KYC / Legal Review: purple
- Negotiation: blue
- Discovery: gray
- Scoping: amber

### Color scheme for keyword tags
- Assign a distinct color per keyword (cycle through: purple, blue, green, amber, pink, teal)

---

## Step 8 — Open the file

```bash
open <output_path>
```

Then report to the user:
- How many opportunities were found
- How many activity threads were found
- The output file path
- Any keywords that returned 0 matches

---

## Known Salesforce Constraints

| Object | Filterable text fields | Not filterable |
|--------|----------------------|----------------|
| Opportunity | Name, StageName | Description, NextStep |
| Task | Subject, Type, Status | Description, Comments |

Because Description is not filterable, the dashboard will only surface records where keywords appear in **Name** (Opportunities) or **Subject** (Tasks). Notes content is fetched separately via WhatId lookup and displayed in the Notes column — it is not used for filtering.

## Error handling

- If a query returns 0 results for all keywords, tell the user and suggest checking the keyword spelling or widening the timeframe.
- If the Task result is saved to a file (>25k tokens), always use Python to parse it — never try to read it directly.
- If `sf org display` fails, ask the user to confirm their default org with `sf config list`.
