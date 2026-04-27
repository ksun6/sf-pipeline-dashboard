---
name: sf-pipeline-dashboard
description: Pull Salesforce opportunities and call reports matching user-specified keywords, then generate an interactive HTML dashboard with notes and SF links. Use when asked to build a pipeline dashboard or search Salesforce for activity around specific topics.
user-invocable: true
allowed-tools:
  - Bash
  - Write
  - mcp__salesforce-dx__get_username
  - mcp__salesforce-dx__run_soql_query
---

# Salesforce Pipeline Dashboard Generator

You are generating an interactive HTML dashboard from live Salesforce data.

## When to Use

Use this skill when the user:
- Asks to "build a pipeline dashboard" or "search Salesforce" for a topic
- Wants to find all opportunities or activity around specific terms (e.g. "CrossX", "ECN")
- Asks for a visual summary of Salesforce deals and call notes matching keywords

## Inputs (from `$ARGUMENTS`)

Parse the following from the user's input:
- **keywords** — one or more comma-separated terms (e.g. "CrossX, ECN, Crossover") — **required**, ask if missing
- **timeframe** — days to look back (default: `365`)
- **limit** — max records per query (default: `500`, max: `2000`)
- **output filename** — default: `dashboard-<keywords>-YYYY-MM-DD.html` where `<keywords>` is the keywords joined by `-` and lowercased (e.g. `dashboard-crossx-ecn-crossover-2026-04-27.html`). Strip spaces and special characters from keyword slugs.

**Keyword sanitization:** Before inserting keywords into any SOQL query, escape single quotes by doubling them (`O'Brien` → `O''Brien`). Strip any characters that are not alphanumeric, spaces, hyphens, or underscores.

---

## Step 1 — Resolve Salesforce org

First, get the current working directory:
```bash
pwd
```
Store as `CWD`. Use this for all MCP `directory` parameters throughout the skill.

Use `mcp__salesforce-dx__get_username` (defaultTargetOrg: true, directory: `CWD`) to get the org username.

Store as `SF_USER`.

---

## Step 2 — Get instance URL

```bash
sf org display --target-org <SF_USER> --json 2>/dev/null | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(d.get('result', {}).get('instanceUrl', '').rstrip('/'))
"
```

Store as `SF_BASE` (e.g. `https://myorg.lightning.force.com`). If empty, ask the user to run `sf config list` and confirm their default org.

---

## Step 3 — Query Opportunities

Build SOQL with one `Name LIKE` clause per keyword joined by `OR`. Apply a `LIMIT` to prevent runaway results.

```sql
SELECT Id, Name, StageName, Amount, CloseDate, Account.Name, Owner.Name, CreatedDate
FROM Opportunity
WHERE CreatedDate >= LAST_N_DAYS:<timeframe>
AND (Name LIKE '%<kw1>%' OR Name LIKE '%<kw2>%' ...)
ORDER BY CreatedDate DESC
LIMIT <limit>
```

**If `totalSize` equals `limit`**, warn the user: _"Results were capped at `<limit>`. Consider narrowing your keywords or reducing the timeframe."_

Collect all returned records as `OPP_RECORDS`.

---

## Step 4 — Query Tasks (Email / Activity Threads + Call Reports)

Run **two queries in parallel**:

**Query A — Subject keyword match** (catches emails + any task with keyword in subject):

```sql
SELECT Id, Subject, Description, ActivityDate, Type, Status, Owner.Name, Account.Name
FROM Task
WHERE ActivityDate >= LAST_N_DAYS:<timeframe>
AND (Subject LIKE '%<kw1>%' OR Subject LIKE '%<kw2>%' ...)
ORDER BY ActivityDate DESC
LIMIT <limit>
```

**Query B — Call/meeting body match** (catches call reports where keyword is only in notes):

```sql
SELECT Id, Subject, Description, ActivityDate, Type, Status, Owner.Name, Account.Name
FROM Task
WHERE ActivityDate >= LAST_N_DAYS:<timeframe>
AND Type IN ('Call', 'Meeting', 'Demo', 'In Person Meeting')
ORDER BY ActivityDate DESC
LIMIT <limit>
```

**Large result handling:** If either result is written to a file (>25k tokens), parse it with Python:

```python
import json, re
with open('<result_file_path>') as f:
    data = json.load(f)
text = data[0]['text']
result = json.loads(text[text.index('{'):])
records = result['records']
total = result['totalSize']
```

If `total` equals `limit` for either query, warn the user the same way as Step 3.

**Post-processing:**
1. Merge Query A and Query B results, deduplicate by `Id`
2. Post-filter: keep only records where `Subject` OR `Description` (case-insensitive) contains any keyword
3. Separate calls/meetings (Type in `['Call','Meeting','Demo','In Person Meeting']` or Subject not starting with `'Email:'`) from email threads
4. Deduplicate email threads: normalize subjects by stripping `Re:`, `RE:`, `Fwd:`, `FW:` prefixes, group by `(Account.Name, normalized_subject[:60])`, keep the most recent per group

Collect as `TASK_RECORDS`.

---

## Step 5 — Pull Notes for Opportunities

Query Tasks related to the found opportunities using `WhatId IN (...)`.

**Batch in groups of 200** — Salesforce enforces a 200-item `IN` clause limit:

```python
opp_ids = [r['Id'] for r in OPP_RECORDS]
batches = [opp_ids[i:i+200] for i in range(0, len(opp_ids), 200)]
```

For each batch, run:

```sql
SELECT Id, Subject, Description, ActivityDate, Type, Owner.Name, WhatId
FROM Task
WHERE WhatId IN ('<id1>','<id2>',...)
AND ActivityDate >= LAST_N_DAYS:<timeframe>
AND Subject NOT LIKE 'Email: Sales Won%'
AND Subject NOT LIKE 'Email: Case%'
AND Subject NOT LIKE 'Email: Case Received%'
ORDER BY WhatId, ActivityDate DESC
LIMIT 500
```

Exclude automated system emails (`Subject LIKE 'Email: Sales Won%'` etc.) — they contain no useful notes.

Merge all batch results. Group by `WhatId` to build: `notes_map = { oppId: [{date, type, owner, text}, ...] }`.

---

## Step 6 — Tag keyword matches

For each opportunity and activity, record which keywords matched:

```python
def tag_keywords(text, keywords):
    t = text.lower()
    return [k for k in keywords if k.lower() in t]
```

- Opportunities: check `Name`
- Tasks: check `Subject` and `Description`

Store as `kw` array on each record (lowercase slugs). Use the original user-provided keyword spelling for display.

---

## Step 7 — Generate the HTML dashboard

Write the output file to `<CWD>/<output_filename>` — always use the current working directory regardless of which folder the skill was invoked from.

The file is self-contained HTML. All filtering and sorting must be client-side JavaScript — no server required.

Embed all data as JavaScript constants at the top of the `<script>` block:

```js
const SF_BASE = "<SF_BASE>";
const KEYWORDS = ["CrossX", "ECN", ...];   // original casing
const OPPS = [ /* opportunity objects */ ];
const EMAILS = [ /* deduplicated task objects */ ];
```

### Required UI elements

**Stat cards (top row)**
- Total Opportunities | Closed Won | In Progress | Closed Lost | Activity Threads
- All update dynamically when filters change

**Filter bar**
- Keyword chips — one per keyword, all active by default; click to toggle
- Free-text search — filters account + name/subject live
- Owner dropdown — populated from unique owners in the data
- Stage dropdown — Opportunities tab only; hide for Activity tab

**Two tabs**
1. **Opportunities** — Date · Account · Opportunity Name · Stage (badge) · Owner · Keywords · Notes · SF Link
2. **Email Activity** — Date · Owner · Account · Subject · Keywords

All columns sortable by clicking the header.

**Notes column**
- No notes → `—`
- Has notes → 1-line truncated preview + "View note" / "View N notes" button
- Button opens a modal with each note in its own block (activity type · date · owner · full text pre-wrapped)
- Modal dismisses on ×, backdrop click, or Escape key

**SF Link column**
- `<a href="<SF_BASE>/<oppId>" target="_blank">↗ Open</a>`

**Stage badge colors**
- Closed Won → green · Closed Lost → red · KYC/Legal → purple
- Negotiation → blue · Discovery → gray · Scoping → amber

**Keyword tag colors** — cycle through: purple, blue, green, amber, pink, teal (one color per keyword, consistent throughout)

---

## Step 8 — Open and report

```bash
open <output_path>
```

Report to the user:
- Opportunities found (and if capped)
- Activity threads found (and if capped)
- Output file path
- Any keywords with 0 matches — suggest checking spelling or broadening timeframe

---

## Known Salesforce Constraints

| Object | Filterable fields | Not filterable |
|--------|------------------|----------------|
| Opportunity | Name, StageName, CloseDate, CreatedDate | Description, NextStep |
| Task | Subject, Type, Status, ActivityDate | Description, Comments |

Because `Description` is not filterable in SOQL, call reports are fetched via a broad `Type IN ('Call','Meeting',...)` query and then post-filtered in Python against `Subject` and `Description`. Email threads are still filtered by `Subject` only. Notes for matched opportunities are fetched separately via `WhatId` lookup.

---

## Error Handling

| Condition | Action |
|-----------|--------|
| No keywords provided | Ask before proceeding |
| Keyword contains `'` | Escape as `''` in SOQL |
| 0 results for all keywords | Report to user; suggest alternate spellings or wider timeframe |
| Results hit `LIMIT` cap | Warn user; suggest narrowing keywords or reducing timeframe |
| Task result > 25k tokens (saved to file) | Parse with Python — never read directly |
| `WhatId IN` batch > 200 IDs | Split into batches of 200 and merge results |
| `sf org display` fails | Ask user to run `sf config list` and confirm default org |
| Salesforce MCP not connected | Tell user to add the Salesforce MCP in Claude Code settings |
