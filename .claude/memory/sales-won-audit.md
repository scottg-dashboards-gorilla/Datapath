# Sales Won Audit — Monthly Process

## Purpose
Validate CRM "Sales Won" data each month-end by cross-referencing against Power BI billing data. Ensures accurate classification of deals as Net New, Upsell, or Renewal — critical for PE presentation where growth vs. non-growth MRR split drives valuation.

## When to Run
- After month-end close, once CRM deals for the month are finalized
- Before updating the dashboard or any PE-facing materials

## Data Sources
1. **CRM Sales Won Report** — Export all closed-won deals for the period. Key fields: Client, MRR, Contract Term, Sales Rep, Deal Type (as tagged by rep)
2. **Power BI Agreement Invoice Comparison** — Shows all active agreements with Previous Invoice Value, Last Invoice Value, delta, and % change. Filter by Agreement Type.
3. **Power BI Agreement Renewals Report** — 2,732 rows, 521 clients (as of 12/27/25). Shows agreement-level detail for cross-referencing client history.

## Classification Rules

### Net New
- Client has ZERO prior agreements in Power BI
- Truly a new logo — never billed before
- CRM often mislabels upsells as "Net New" — always verify

### Upsell (Growth)
- Client has existing agreements in Power BI
- Deal adds new service lines, products, or increases scope
- Includes: service additions, HW/product subscriptions, security add-ons, M365 licenses, user count increases
- **Resale deals are Upsell** unless they replace existing recurring MRR that already existed (replacement = Renewal)
- **Other/Misc deals are Upsell** if client is existing (confirmed in Power BI)

### Renewal
- Client renewing an existing agreement at the same or similar rate
- No significant new services added
- Price adjustments within normal range (inflation, minor scope changes)
- **If a "renewal" includes significant new services or price increases beyond normal adjustment, it may be an Upsell**

### Key Rule: Resale Classification
> "Resale would be growth unless it is a replacement of recurring MRR revenue that already existed."
- Check Power BI for existing Recurring Resale agreements (MSP, Security, M365 NCE)
- If the new resale deal adds a product line the client didn't have → Growth (Upsell)
- If the new resale deal replaces an expiring product subscription → Renewal (not growth)
- Evidence: existing agreements continue billing alongside new ones = Growth

## Step-by-Step Process

### Step 1: Export CRM Data
- Pull all closed-won deals for the audit period
- Columns needed: Close Date, Client, MRR Won, Contract Term, Sales Rep, Deal Type, Notes

### Step 2: Cross-Reference Net New Claims
- For every deal tagged "Net New" in CRM:
  1. Search client name in Power BI Agreement Renewals report
  2. If client has existing agreements → reclassify as Upsell
  3. If client has zero agreements → confirmed Net New
  4. Record prior agreement MRR for context

### Step 3: Audit Resale Deals
- Filter Power BI to Recurring Resale agreement types (MSP, Security, M365 NCE)
- For each resale deal:
  1. Check if client has existing resale agreements
  2. Check if the new deal replaces an expiring agreement (same product type, similar value)
  3. If no replacement evidence → classify as Growth (Upsell)
  4. If replacement → classify as Renewal

### Step 4: Audit Other/Misc Deals
- For each "Other" deal:
  1. Search client in Power BI
  2. If client has existing agreements → Upsell (additional service to existing client)
  3. If client NOT found → may be Net New (investigate further)
  4. Note: "Other" was historically a catch-all for deals that didn't fit original classification

### Step 5: Audit Renewals for Growth Signals
- For each renewal deal:
  1. Pull client's agreement history from Power BI
  2. Compare Last Invoice Value to Previous Invoice Value
  3. If significant positive delta (+10% or more, or new service lines added) → consider reclassifying as Upsell
  4. If significant negative delta → note as contraction risk
  5. If flat or minor change → confirmed Renewal

### Step 6: Update Deliverables
1. **Spreadsheet** (`Sales Won Audit - MRR Analysis.xlsx`):
   - Update Category column for reclassified deals
   - Update Is Growth? column (YES for Net New + Upsell)
   - Add Notes explaining reclassification rationale
   - Update Summary sheet totals
   - Update By Sales Rep sheet
2. **Dashboard** (`monthly financial analysis.html`):
   - Update ALL_DEALS array entries (change `cat:` values)
   - Verify KPI card totals recalculate correctly
   - Update footer audit date

### Step 7: Verify Consistency
- Cross-check: Spreadsheet totals must match dashboard totals
- Net New deals + Upsell deals = Growth MRR
- Growth % = (Net New + Upsell) / Total MRR Won
- Deal counts by category must match across both files

## Power BI Navigation Notes

### Agreement Invoice Comparison
- Located under Reports in Power BI workspace
- Has two filter mechanisms:
  1. **Agr Location** filter in the filter panel (left side)
  2. **Agreement Location** slicer/dropdown at top of report
  - BOTH must be set correctly — use eraser icon to "Clear selections" for All locations
- Agreement Type dropdown has checkboxes for filtering by type
- Table uses virtual scrolling — only ~20-25 rows visible at a time in DOM
- DOM structure: `[role="gridcell"]` for all columns, Cell[0] = checkbox, Cell[1] = Customer name
- Cell values contain "Additional Conditional Formatting" text that must be cleaned

### Data Extraction Tips (UI scraping — SLOW, avoid)
- Extract in batches due to virtual scrolling
- Use shorter delimiters to avoid output truncation
- Multiple scroll+extract cycles needed for full dataset
- Clean extracted data: remove "Additional Conditional Formatting" text, trim whitespace

### Power BI REST API — FAST, use this instead
The Power BI API lets you run DAX queries directly against the dataset, bypassing the UI entirely. This is **dramatically faster** than scraping the virtual-scroll table.

#### Setup (run from Chrome DevTools / javascript_tool on any Power BI page)
```javascript
// These values are stable across sessions:
const token = window.powerBIAccessToken;  // auto-refreshed by Power BI session
const groupId = 'f17eba89-44a2-4b3d-94a7-1d2ef4e84358';
const datasetId = '5ee15556-ded0-4d30-b34b-21757d0c1a82';
```

#### Key Tables in the Dataset (33 total, these matter)
| Table | Purpose |
|-------|---------|
| `reportdimension Company` | Customer name, status, territory, account manager |
| `reportdimension Agreement` | Agreement name, type, category, status, billing cycle |
| `reportfact Agreements` | **StartDate**, EndDate, CancelDate, Value, Agreement First Billing Date, Agreement Term |
| `reportfact Billings` | Invoice-level billing data |
| `reportdimension Billing` | Billing dimensions |
| `reportdimension AgreementAddition` | Agreement addition details |
| `reportfact AgreementAdditions` | Addition fact data |
| `reportdimension BusinessUnit` | Business unit/department |
| `core Organisation` | Organization-level data |

#### DAX Query: Earliest Agreement Start Date per Customer (the key query)
```javascript
(async () => {
  const token = window.powerBIAccessToken;
  const groupId = 'f17eba89-44a2-4b3d-94a7-1d2ef4e84358';
  const datasetId = '5ee15556-ded0-4d30-b34b-21757d0c1a82';
  
  const query = `
    EVALUATE
    SUMMARIZE(
      'reportfact Agreements',
      'reportdimension Company'[Customer],
      "EarliestStart", MIN('reportfact Agreements'[StartDate]),
      "EarliestBilling", MIN('reportfact Agreements'[Agreement First Billing Date]),
      "AgreementCount", COUNTROWS('reportfact Agreements')
    )
    ORDER BY 'reportdimension Company'[Customer]
  `;
  
  const resp = await fetch(
    `https://api.powerbi.com/v1.0/myorg/groups/${groupId}/datasets/${datasetId}/executeQueries`,
    {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${token}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({ queries: [{ query }], serializerSettings: { includeNulls: true } })
    }
  );
  const data = await resp.json();
  const rows = data.results?.[0]?.tables?.[0]?.rows || [];
  // rows[i] keys: 'reportdimension Company[Customer]', '[EarliestStart]', '[EarliestBilling]', '[AgreementCount]'
  // Date format: '2025-03-01T00:00:00'
  return JSON.stringify({ count: rows.length, sample: rows.slice(0, 3) });
})();
```

#### Other Useful DAX Queries
```sql
-- All agreements for a specific customer
EVALUATE
FILTER(
  SUMMARIZE(
    'reportfact Agreements',
    'reportdimension Company'[Customer],
    'reportdimension Agreement'[Agreement Name],
    'reportdimension Agreement'[Agreement Type],
    "Start", MIN('reportfact Agreements'[StartDate]),
    "End", MIN('reportfact Agreements'[EndDate]),
    "Value", SUM('reportfact Agreements'[Value])
  ),
  'reportdimension Company'[Customer] = "Advantage Home Care"
)

-- Simple connectivity test
EVALUATE ROW("test", 1)

-- Discover all columns in any table
EVALUATE TOPN(1, 'reportfact Agreements')

-- List all tables and columns
EVALUATE COLUMNSTATISTICS()
```

#### Important Notes
- `powerBIAccessToken` is available on any Power BI page — just navigate to the report first
- Token refreshes automatically with the session; no manual auth needed
- Dataset ID is stable (tied to the workspace, not the report)
- Returns up to ~100K rows per query; use TOPN() or FILTER() for large results
- Wrap in `(async () => { ... })()` for javascript_tool since top-level await isn't supported

## Current Audit Results (as of 4/26/2026)

### Category Totals (204 deals, Jan 2025 – Feb 2026)
| Category | Deals | MRR/mo |
|----------|-------|--------|
| Net New | 28 | $60,963 |
| Upsell | 140 | $176,176 |
| Renewal | 36 | $97,964 |
| **Total** | **204** | **$335,103** |

### Growth Split
- Growth (Net New + Upsell): $237,139/mo (70.8%)
- Non-Growth (Renewal): $97,964/mo (29.2%)
- Net New logos: 20 unique clients

### Net New Clients (20 logos, first Power BI agreement 2025+)
- Advantage Home Care (2025-03-01), Ameribest (2025-01-31), Yosemite Farm Credit (2025-04-01)
- PASCO (2025-01-23), Impact Home Health (2025-03-01), Inteli-care LLC (2025-04-01)
- City of Mendota (2025-04-18), Samaritan Home Health (2025-09-01), Jame O Bookkeeping (2025-09-01)
- ECS (2025-10-01), Cerbat Mountain Farm LLC (2025-10-01), Rizo Lopez Foods (not in Power BI)
- Sierra Northern Railway (2026-03-01), The Ohio District - LCMS (2026-03-01), Sierra Energy (2026-03-01)
- Central Valley Regional Center, Statuo, Mendocino Railway, Advanced Radiology, Railpower (2026 logos)

### Reclassification History
- 29 Resale deals ($23,183/mo) → all reclassified as Upsell (net new product subscriptions, not replacements)
- 37 Other/Misc deals ($18,688/mo) → all reclassified as Upsell (additional services to existing clients)
- 1 Renewal (Imagine Columbus Primary Academy, $229/mo) → reclassified as Upsell (new service add confirmed in Power BI)
- 21 Upsell deals ($41,014/mo) across 13 clients → reclassified as Net New (Power BI API query confirmed first agreements 2025+)
- Method: DAX query via Power BI REST API against dataset 5ee15556, table 'reportfact Agreements', MIN(StartDate) per customer

## Files
| File | Path |
|------|------|
| Sales Won Spreadsheet | `dashboards/Sales Won Audit - MRR Analysis.xlsx` |
| Dashboard (Sales Won tab) | `dashboards/monthly financial analysis.html` → `renderSalesWon()` |
| Resale Audit Detail | `dashboards/Resale Audit - Growth Reclassification.md` |
| Other/Misc Audit Detail | `dashboards/Other-Misc Audit - Reclassification.md` |
| Renewal Audit Detail | `dashboards/Renewal Audit - Power BI Analysis.md` |
