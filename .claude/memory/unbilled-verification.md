# Monthly Unbilled MRR Verification Process

## Overview
Each month after Sales Won data is pulled from CRM (Power BI), all new MRR deals must be cross-referenced against ConnectWise (CW) agreements to determine which are actually being billed. Until verified, deals are added to the `UNBILLED_DEALS` array in the dashboard with `risk:'med'` status.

## When to Run
- **Timing**: After the close of each month, once CRM Sales Won data is available
- **Frequency**: Monthly
- **Prerequisite**: March or later month's deals have been added to `ALL_DEALS` in the dashboard

## Step-by-Step Process

### 1. Pull New Deals from CRM
- Navigate to Power BI Sales Won report
- Filter to the target month
- Export or manually note all MRR deals (client, amount, type, rep, close date)

### 2. Add Deals to UNBILLED_DEALS Array
- Add every new month's MRR deal to `UNBILLED_DEALS` in both:
  - `monthly financial analysis.html` (main dashboard)
  - `index.html` (GitHub Pages mirror)
- Format: `{close:'Mon YYYY',c:'Client Name',mrr:AMOUNT,type:'MRR - ...',rep:'Rep Name',status:'🆕 Verify addition on existing agreement',days:XX,risk:'med'}`
- Special cases:
  - Contract replacements: use `status:'🔄 [description] — verify new agreement'`
  - Net New logos with no CW history: use `status:'🆕 No agreement history'`, `risk:'high'`

### 3. Verify Against ConnectWise
For each deal in UNBILLED_DEALS:

#### For Upsells / Existing Clients:
1. Search client in ConnectWise → Finance → Agreements
2. Find the active Managed Services agreement
3. Go to **Additions** tab (not just the parent agreement amount)
4. Sort by **Effective Date** descending
5. Look for a new addition matching the CRM deal amount, effective around the deal close date
6. **Key check**: The addition amount should match (or closely match) the CRM MRR amount

#### For Net New Logos:
1. Search client in ConnectWise
2. Check if ANY agreement exists
3. If no agreement at all → `risk:'high'`, status stays `'🆕 No agreement history'`
4. If agreement exists, check Additions for the specific deal amount

#### For Contract Replacements (like expired M365 renewals):
1. Find the client's existing agreement
2. Check if a NEW agreement was created (or addition added) to replace the expired contract
3. The old agreement may show as expired/cancelled; the new one should cover the CRM amount

### 4. Update Statuses After Verification
- **Verified billing**: Remove from `UNBILLED_DEALS` (deal stays in `ALL_DEALS`)
- **Addition found but amount doesn't match**: Update status to note discrepancy, keep in array
- **No addition found**: Escalate — the deal was won in CRM but never set up for invoicing
  - Update `risk:'high'`
  - Update status to `'⚠️ No CW addition found — escalate to ops'`

### 5. Bulk Verification Shortcut
Instead of checking client-by-client:
1. In ConnectWise → Finance → Agreements → search across all
2. Go to any agreement's Additions tab
3. Sort the **Effective Date** column descending
4. Scan for additions with dates in the target month
5. Cross-reference against the UNBILLED_DEALS list

### 6. Dashboard Sync
After all updates:
- Both `monthly financial analysis.html` and `index.html` must be updated identically
- Provide Claude Code sync instructions for commit/push

## ConnectWise Navigation Notes
- URL pattern: `https://na.myconnectwise.net/v2025_2/...`
- Agreements: Finance → Agreements → search by company name
- Additions tab: Inside each agreement, click "Additions" to see line items
- Effective Date filter: Type date or sort column to find recent additions
- The CW agreement total may not match CRM MRR exactly (CW includes all line items, CRM tracks individual deals)

## Status Icons Reference
| Icon | Meaning |
|------|---------|
| 🆕 | New deal, needs initial verification |
| 🔄 | Contract replacement (e.g., expired license renewal) |
| ⚠️ | Problem found (expired agreement, no CW match, etc.) |
| ✅ | Verified — ready to remove from UNBILLED_DEALS |

## Risk Levels
| Level | Meaning |
|-------|---------|
| `high` | Net new logo with no CW history, or verified billing gap |
| `med` | Existing client, awaiting addition verification |
| `low` | Minor discrepancy, likely timing issue |
