---
name: sales-won-audit
description: >
  Monthly Sales Won audit — pull new CRM deals from Power BI, categorize them as Net New / Upsell / Renewal,
  cross-reference against ConnectWise and Power BI billing data, and update the dashboard and spreadsheet.
  Use this skill whenever the user asks to: pull new sales won data, categorize deals, audit sales categories,
  run the monthly sales won process, add new month's deals to the dashboard, classify deals as net new vs upsell,
  check if deals are renewals or growth, update the sales won view, reconcile CRM vs billing data, or anything
  related to Sales Won analysis. Also use when the user says "pull this month's deals", "categorize the new deals",
  "update sales won", "run the sales audit", "what's net new vs upsell", or "add March deals to the dashboard".
---

# Sales Won Audit — Monthly Process

## Purpose

Each month, new "Sales Won" deals close in the CRM and appear in Power BI. These deals need to be categorized correctly as Net New, Upsell, or Renewal — because the growth vs. non-growth MRR split directly drives Datapath's PE valuation. CRM reps frequently miscategorize deals (tagging upsells as "Net New," for example), so every deal must be verified against Power BI billing data.

This skill covers the full lifecycle: pulling the data, categorizing it, cross-referencing, updating the dashboard (`ALL_DEALS` array) and spreadsheet, and flagging any unbilled deals for the `unbilled-mrr-verification` skill.

## When to Run

- **Monthly**: After month-end close, once CRM deals are finalized (typically by the 5th)
- **On demand**: When the user asks to audit or reclassify specific deals
- **Before PE reporting**: Always run before updating any investor-facing materials

## Data Sources

1. **Power BI — Sales Rep Quota Tracking**: CRM deals with close date, client, MRR, contract term, sales rep, deal type
2. **Power BI — Agreement Invoice Comparison**: Active agreements with Previous Invoice Value, Last Invoice Value, delta, and % change. Used to verify whether a client is truly "net new" or has existing billing
3. **Power BI — Agreement Renewals Report**: 2,732+ agreements across 521+ clients. The definitive source for whether a client has prior Datapath agreements
4. **ConnectWise**: Agreement details, additions, and invoice history (used by the `unbilled-mrr-verification` skill)

## Classification Rules

Understanding these rules is critical — they determine how PE buyers evaluate Datapath's growth story.

### Net New (truly new logos)
- Client has **zero** prior agreements in Power BI
- Never been billed by Datapath before
- CRM frequently mislabels existing-client upsells as "Net New" — always verify against Power BI
- Only ~7 out of 200+ deals were genuinely Net New in the Jan 2025–Mar 2026 audit

### Upsell (growth from existing clients)
- Client has existing agreements in Power BI
- Deal adds new service lines, products, or increases scope
- Includes: managed service additions, security add-ons, M365 licenses, user count increases, hardware subscriptions
- **Resale deals default to Upsell** unless they replace recurring MRR that already existed (replacement = Renewal)
- **Other/Misc deals default to Upsell** if the client exists in Power BI

### Renewal (retention, not growth)
- Client renewing an existing agreement at the same or similar rate
- No significant new services added
- Price adjustments within normal range (inflation, minor scope changes)
- If a "renewal" includes significant new services or a price increase beyond normal adjustment (+10% or more), consider reclassifying as Upsell

### The Resale Rule
> "Resale is growth unless it replaces recurring MRR revenue that already existed."

Check Power BI for existing Recurring Resale agreements. If the new resale deal adds a product the client didn't have, it's growth (Upsell). If it replaces an expiring subscription of the same type, it's Renewal.

## Step-by-Step Workflow

### Phase 1: Pull New Month's Data from Power BI

1. Open the **Sales Rep Quota Tracking** report in Power BI (Chrome tab)
2. Use Claude in Chrome MCP tools to navigate and extract data
3. Filter to the target month's closed-won deals
4. Extract all deals with these fields:
   - Close Date / Month
   - Client Name
   - MRR Won
   - Contract Term (months)
   - Sales Rep
   - Deal Type (as tagged in CRM)
5. Present the raw deals to the user for initial review

**Power BI extraction tips:**
- The table uses virtual scrolling — only ~20-25 rows visible at a time
- Extract in batches: read visible rows, scroll down, extract next batch
- DOM cells contain "Additional Conditional Formatting" text — clean this out
- Multiple scroll+extract cycles are needed for the full dataset

### Phase 2: Categorize Each Deal

For every deal pulled from Power BI:

1. **Search the client name** in the Agreement Invoice Comparison or Agreement Renewals report
2. **If client has zero agreements**: Mark as **Net New** (genuine new logo)
3. **If client has existing agreements**: Mark as **Upsell** (growth from existing client)
4. **If deal type is "Renewal"**: Check Power BI for price delta:
   - Flat or minor change → confirmed **Renewal**
   - Significant increase (+10%+) or new service lines → reclassify as **Upsell**
5. **If deal type is "Resale"**: Check for existing resale agreements:
   - No prior resale agreement of same type → **Upsell** (growth)
   - Replaces expiring agreement → **Renewal**
6. **Record `prior` MRR** for each deal (the client's existing agreement MRR before this deal)

### Phase 3: Build the ALL_DEALS Entries

Format each deal for the dashboard's `ALL_DEALS` array:

```javascript
{m:'Mar 2026', c:'Client Name', mrr:1500, cy:12, r:'Sales Rep', cat:'Upsell', prior:3200}
```

Fields:
- `m`: Month and year (e.g., `'Mar 2026'`)
- `c`: Client name (must match CRM exactly)
- `mrr`: Monthly recurring revenue (number, no $)
- `cy`: Contract term in months (12, 24, 36, etc.)
- `r`: Sales rep name
- `cat`: Category — one of `'Net New'`, `'Upsell'`, `'Renewal'`
- `prior`: Client's existing MRR before this deal (0 for Net New)

### Phase 4: Identify Unbilled Deals

Any deal that was recently closed but might not yet be set up in ConnectWise should be added to the `UNBILLED_DEALS` array for verification:

```javascript
{close:'Mar 2026', c:'Client Name', mrr:1500, type:'MRR - Managed Services (Upsell)', rep:'Sales Rep', status:'New — needs verification', days:26, risk:'med'}
```

- Net New deals are **high priority** for unbilled verification (brand new client = no existing billing)
- Upsells on existing clients are **medium priority** (billing infrastructure exists but addition may not be set up)
- Renewals are **low priority** (existing agreement likely auto-renews)

After adding to UNBILLED_DEALS, use the `unbilled-mrr-verification` skill to verify against ConnectWise.

### Phase 5: Update Dashboard

1. Add new entries to the `ALL_DEALS` array in `monthly financial analysis.html`
2. Sort entries by month (newest last) within the array
3. Update the section subtitle with the new date range and deal count
4. Verify KPI card totals recalculate correctly (Growth MRR, Renewal MRR, Total MRR)
5. Apply the **same changes** to `index.html` (GitHub Pages mirror)

### Phase 6: Update Spreadsheet

Update `Sales Won Audit - MRR Analysis.xlsx`:
1. Add new rows for each deal on the Detail sheet
2. Update the Category column with verified classifications
3. Update the Is Growth? column (YES for Net New + Upsell, NO for Renewal)
4. Add Notes explaining classification rationale where applicable
5. Update Summary sheet totals
6. Update By Sales Rep sheet

### Phase 7: Verify Consistency

Cross-check that everything matches:
- Spreadsheet totals must match dashboard totals
- Net New + Upsell = Growth MRR
- Growth % = (Net New + Upsell) / Total MRR Won
- Deal counts by category must match across both files
- No duplicate deals (same client + same MRR + same month = likely duplicate)

### Phase 8: Report and Sync

Present a summary showing:
- New deals added (count and total MRR)
- Category breakdown: Net New / Upsell / Renewal
- Growth vs Non-Growth split
- Any reclassifications from CRM's original categorization
- Deals added to UNBILLED_DEALS for CW verification
- Claude Code sync instructions for commit/push

## Dashboard Data Structure

### ALL_DEALS Array
Located at line ~4440 in `monthly financial analysis.html`. Contains all closed-won deals across the audit period.

### UNBILLED_DEALS Array
Located at line ~4681. Contains deals that need ConnectWise billing verification. Managed by the `unbilled-mrr-verification` skill.

### File Sync Rule
Both files must always be updated identically:
- `monthly financial analysis.html` (main dashboard)
- `index.html` (GitHub Pages mirror)

## Key Files

| File | Path |
|------|------|
| Dashboard (main) | `monthly financial analysis.html` |
| Dashboard (GitHub) | `index.html` |
| Sales Won Spreadsheet | `Sales Won Audit - MRR Analysis.xlsx` |
| Memory (audit process) | `.claude/memory/sales-won-audit.md` |
| Unbilled verification skill | `.claude/skills/unbilled-mrr-verification/SKILL.md` |

## Common Pitfalls

- **CRM "Net New" is usually wrong**: In the Jan 2025–Feb 2026 audit, ~94 out of ~100 "Net New" tagged deals were actually Upsells to existing clients. Always verify against Power BI.
- **Resale is not a final category**: Every Resale deal should be reclassified as either Upsell (growth) or Renewal (retention). The dashboard doesn't show a Resale category in the growth calculation.
- **"Other/Misc" needs reclassification too**: These were a catch-all. Almost always Upsell if the client exists in Power BI.
- **Prior MRR matters**: Record the `prior` field for each deal — it shows what the client was already paying, which helps PE buyers understand the growth trajectory.
- **Contract terms vary**: Don't assume 12 months. Check the actual CY value — some deals are 24 or 36 month commitments.
- **Month format must be consistent**: Use `'Mon YYYY'` format (e.g., `'Mar 2026'`) — this drives the dashboard's monthly grouping and sorting.
- **Don't double-count**: If a client has multiple deals in the same month (e.g., managed services + security), they should be separate entries, not combined.

## Historical Context

As of the Mar 2026 audit (264 deals, Jan 2025–Mar 2026):
- 7 confirmed Net New logos
- ~200+ Upsell deals (includes reclassified Resale, Other, and false "Net New")
- 36 Renewals
- Growth MRR (Net New + Upsell) represents ~70% of total MRR Won
- This growth-heavy mix is a strong PE signal
