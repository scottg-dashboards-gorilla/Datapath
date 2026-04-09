# Dashboard Details & Functions

## Dashboard Views (7 tabs)
1. **Dashboard** — KPIs, charts, intelligence report with anomalies below, Products Ordered Not Invoiced table
2. **Departments** — Drill-down cards per department
3. **Gross Profit** — Product-level Rev/COGS/Net pairing with filter tabs
4. **P&L Detail** — Full P&L with labor allocation reference
5. **Labor Allocation** — Per-person dept allocation with collapsible sections (CONFIDENTIAL)
6. **Intelligence** — Two-panel: anomalies timeline + AI advisor chat (Claude API)
7. **Import Data** — Upload modal for new P&L files

## Column Layout (GP and P&L views)
3 prior months | Current (bold) | MOM Delta | MOM % | 3-MO AVG | VS AVG Delta | VS AVG % | L12M BEST (month label) | VS L12M Delta | VS L12M % | ATH (month label) | VS ATH Delta | VS ATH %

## Key Functions
- `makeMonth()` — builds monthly data; supports legacy (direct/project) and new (byDept) labor allocation
- `getDeptDetailLines()` — per-department revenue/COGS with labor
- `pnlTableEngine()` — shared table engine for GP and P&L views
- `renderGrossProfit()` — GP view with filter tabs and product pairing
- `renderPnL()` — full P&L detail view
- `renderLaborAllocation()` — per-person labor view with collapsible dept blocks
- `buildAnomalyHtml()` — anomaly timeline for Intelligence panel
- `renderAdvisor()` — two-panel Intelligence view
- `renderSNITable()` / `sortSNI()` / `filterSNI()` — SNI customer table handlers
- `l12Best(getter, isCost)` — L12M BEST: highest for revenue, lowest for costs
- `ath(getter)` — All-Time High (always highest)
- `fDelta()` / `fPct()` — accept `invert` parameter for cost rows

## Data Objects
- `DATA` array — 38 monthly entries (Jan23-Feb26)
- `ANOMALIES` object — monthly anomaly notes (all months Jan23-Feb26)
- `WORKING_DAYS` object — working days per month (federal holidays excluded)
- `SNI_CUSTOMERS` array — 49 customers, 254 items, 84 orders
- `SNI_TOTALS` — Rev $631K, Cost $531K, Profit $100K, Margin 15.8%
- `ATH_EXCLUDE` = `['Dec 25']`
- `DATAPATH_SYSTEM_PROMPT` — full business context for Claude API advisor
- `conversationHistory` — multi-turn context within advisor session

## MSP Strategy Advisor (Intelligence Tab)
- Powered by Claude API (claude-sonnet-4-20250514)
- API key in localStorage (`datapath_anthropic_key`) + JS variable
- User chose client-side key approach, plans MFA via GitHub
- `connectApiKey()` / `disconnectApiKey()` manage lifecycle
- `buildFinancialSnapshot()` generates live data for each API call
- POST to `https://api.anthropic.com/v1/messages` with `anthropic-dangerous-direct-browser-access: true`
- Quick buttons: EBITDA, Valuation, Labor, Security, Benchmarks, Cloud, Add-backs, R&D
- Budget recommendation: $10-25/mo at console.anthropic.com

## Products Ordered Not Invoiced (SNI)
- Source: "Products Ordered not Invoiced 3.6.2026.xlsx" (date range 9/1/2025 – 2/28/2026)
- Top customers: Newman-Crows Landing USD ($139K), Advanced Radiology ($83K), Vital Care ($71K), Scheurer Health ($66K)
- Table: sortable by column click, searchable by customer name
- KPI cards: Total Sell Price, Cost, Gross Profit, Blended Margin
- Power BI reference URL available in dashboard code

## Dashboard Roadmap (Intelligence tab bottom)
- 2-column grid layout (was 3, removed one card)
- Remaining cards: "Won Revenue Not Yet Invoiced" and "New MRR Won — Not Yet Onboarded"
- Removed: "Department-Level Resource Allocation" (user updated externally)

## GP View
- Revenue → COGS → Net for each product line
- Labor COGS tagged with `parent` field to group under revenue line
- ALLOC badge for labor items, NEEDS DATA for missing COGS

## GitHub Deployment
- Username: `scottg-dashboards-gorilla`
- Repo: `Datapath` (private)
- Pages URL: `https://scottg-dashboards-gorilla.github.io/Datapath/`
- PAT: `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`
- Scoped to Datapath repo only (Contents R/W, Metadata R/O)
- Created: March 29, 2026 | Expires: ~June 26, 2026

## Data Sources
- 2023: "2023 monthly details.xlsx" (Combined Company tab) — payroll estimated split (47.8/6.8/20.5/24.9%)
- 2024: "Budget vs Actual 2024 - January to December.xlsx" (All, CA, OH tabs) — security embedded in MS
- 2025 Jan-May: Individual Citrin Cooperman P&L files with tabs (P&L, Footnote, Payroll Breakdown, Agreements)
- 2025 Jun-Dec: From consolidated P&L exports
- 2026 Jan-Feb: Individual Citrin Cooperman monthly P&L files

## User Preferences
- NO "Division 1, 2" — use actual department names
- NO staff counts in labels or headcount KPI card
- Light/white theme (#f8f9fa, #ffffff)
- Anomalies displayed on main dashboard below intelligence report
- Working days row at top of GP and P&L tables
- COGS - Managed IT labeled as "Tech Stack" costs
