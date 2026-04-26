# Datapath Financial Intelligence Platform

## Project Overview
Datapath is a 25-year-old MSP (Managed Service Provider) headquartered in Modesto, CA with operations in Ohio. The company is preparing for a PE exit at 8-11x adjusted EBITDA ($30M-$35M target). This project consists of three deliverables:

1. **Financial Dashboard** — Single-file HTML app (38 months of P&L data, Jan23-Feb26) with Chart.js visualizations, Claude API-powered advisor, and interactive P&L/GP/labor views
2. **PE Investment Memorandum** — DOCX for prospective PE buyers with financial narrative and EBITDA bridge
3. **P&L to EBITDA Spreadsheet** — Monthly 2025 P&L with 19-item addback schedule showing Adjusted EBITDA

Key people: Co-CEOs James Bates & David Darmstandler, COO Scott Gordon (consultant via CLevelAdvising LLC, $25,500/mo — CONFIDENTIAL, never break out individually).

## Tech Stack
- **Dashboard**: Vanilla JS + Chart.js + SheetJS, single HTML file, SHA-256 password gate (password: `G0dz1ll@`)
- **DOCX generation**: `docx` npm module (v304). Requires `NODE_PATH=/usr/local/lib/node_modules_global/lib/node_modules`
- **Excel**: openpyxl (Python). Recalc script: `python3 /sessions/dazzling-awesome-bell/mnt/.claude/skills/xlsx/scripts/recalc.py` — run TWICE for cascading formulas
- **DOCX tools**: unpack/pack/validate scripts in `/sessions/dazzling-awesome-bell/mnt/.claude/skills/docx/scripts/office/`
- **GitHub Pages**: User `scottg-dashboards-gorilla`, repo `Datapath`, PAT expires ~June 26 2026
- **Claude API**: claude-sonnet-4-20250514 in dashboard advisor, key in localStorage, `anthropic-dangerous-direct-browser-access` header
- **Python pip**: Always use `--break-system-packages` flag

## Coding Conventions
- **P&L structure**: "Total Operating Net Income - DL above the line" = EBITDA line; partnership expenses are BELOW this line
- **Labor allocation**: Legacy model (2025-) puts all direct labor into MS COGS. New model (Jan26+) distributes by dept using `labor.byDept` parameter
- **Color logic**: Revenue rows — higher=green. Cost rows (`isCost:true`) — LOWER=green (inverted). Use `fDelta()`/`fPct()` with `invert` param
- **Dashboard deploy**: `index.html` is auto-synced copy of `monthly financial analysis.html` via scheduled task (every 10 min)
- **Excel PermissionError workaround**: Save to temp location first, then `cp` to dashboards folder
- **Addback formulas**: Monthly Total = `SUM(B84:B100)` (excludes Facilities rows 82-83). Annual Total = `SUM(N82:N100)` (includes Facilities). This is intentional per user's file structure
- **ATH_EXCLUDE**: `['Dec 25']` — year-end months excluded from ATH/L12M benchmarks
- **Department names**: Never use "Division 1, 2" — always actual department names
- **No headcount**: Never show staff counts in labels or KPI cards

## What to Avoid
- **NEVER regenerate PE book from pe-book.js** — User's uploaded narrative is the MASTER. Only use XML unpack/edit/repack workflow
- **NEVER break out Scott Gordon's compensation individually** — Include only in aggregate Sr Leadership totals
- **NEVER use `italics=True`** in openpyxl — correct keyword is `italic=True`
- **NEVER clear merged cells directly** — call `ws.unmerge_cells()` before setting values on merged ranges
- **Don't add "Other Income / (Expense)" to EBITDA** — it's below the line (co-CEO charges, stockholder expenses, interest)
- **Don't assume 2023-2024 has dept-level security revenue** — it's embedded in managed_services for those years
- **Don't assume 2023 payroll is split** — it's all-in; use estimated ratios (direct 47.8%, project 6.8%, sales 20.5%, admin 24.9%)

## Context Pointers
- **Full financial data** (revenue, EBITDA, addback schedule, P&L structure): Read `.claude/memory/financials.md`
- **Employee roster & personnel changes**: Read `.claude/memory/employees.md`
- **Dashboard details** (views, functions, anomalies, SNI data, API): Read `.claude/memory/dashboard.md`
- **PE Investment Memorandum** (sections, editing workflow, narrative rules): Read `.claude/memory/pe-book.md`
- **Legacy detailed context** (all-in-one reference): `projects/-sessions-dazzling-awesome-bell/memory/datapath-context.md`

## Key File Paths
| File | Path |
|------|------|
| Dashboard (main) | `dashboards/monthly financial analysis.html` |
| Dashboard (GitHub) | `dashboards/index.html` |
| PE Investment Memorandum | `dashboards/Datapath - Confidential Investment Memorandum.docx` |
| P&L to EBITDA | `dashboards/Datapath 2025 P&L to EBITDA.xlsx` |
| Sales Won Audit | `dashboards/Sales Won Audit - MRR Analysis.xlsx` |
| P&L Imports | `dashboards/P&L Imports/` (new Citrin xlsx go here) |
| Weekly SNI snapshots | `dashboards/weekly-snapshots.xlsx` |

## Scheduled Tasks
- **weekly-sni-refresh**: Mondays 9am — pulls SNI data from Power BI
- **push-dashboard-to-github**: Every 10 min (8am-8pm) — pushes if local file changed
- **github-token-expiry-reminder**: One-time June 21, 2026

## Key Numbers (2025)
- Revenue: $17,737,413 | COGS: $11,092,086 | Gross Profit: $6,645,327
- OpEx: $3,293,096 | EBITDA: $1,595,160
- Total Addbacks: $1,297,247 (19 items) | Adjusted EBITDA: $2,892,407 | Margin: 16.3%
- Headcount: 58 (as of 3/31/2026)
