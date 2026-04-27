# sf-pipeline-dashboard

A Claude Code skill that queries Salesforce for opportunities and call reports matching user-specified keywords, then generates a self-contained interactive HTML dashboard.

## What it does

1. Resolves your default Salesforce org via `sf` CLI
2. Searches Opportunities by `Name` and Tasks by `Subject` for the given keywords
3. Fetches related call/meeting notes for each opportunity
4. Generates a single HTML file with:
   - Summary stat cards (Won / In Progress / Lost / Activities)
   - Keyword filter chips, search, owner + stage dropdowns
   - Opportunities tab with stage badges, notes modal, and direct SF links
   - Email/Activity tab with deduplicated conversation threads
   - All filtering and sorting client-side — no server required

## Installation

### Option A — Global (available in any Claude Code session)
```bash
mkdir -p ~/.claude/skills/sf-pipeline-dashboard
cp .claude/skills/sf-pipeline-dashboard/SKILL.md ~/.claude/skills/sf-pipeline-dashboard/SKILL.md
```

### Option B — Project-local
Copy `.claude/` into your project directory. Claude Code will auto-load it when that directory is the workspace root.

## Usage

```
/sf-pipeline-dashboard CrossX, ECN, Crossover
```

With a custom timeframe (days):
```
/sf-pipeline-dashboard keywords: "OES, DMA" timeframe: 180
```

## Requirements

- [Claude Code](https://claude.ai/code)
- [Salesforce CLI (`sf`)](https://developer.salesforce.com/tools/salesforcecli) with a configured default org
- Salesforce MCP server connected in Claude Code settings

## Example output

See [`example-dashboard.html`](./example-dashboard.html) — a dashboard built from a real Salesforce query for CrossX / ECN / Crossover activity over 12 months (anonymized).
