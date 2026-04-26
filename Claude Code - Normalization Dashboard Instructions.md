# Claude Code: Add EBITDA Normalization to Dashboard

## What Changed
Cowork added an EBITDA Normalization Framework to `.claude/memory/financials.md`. This tracks one-time items *above* the EBITDA line — separate from owner addbacks. PE buyers need to see: **Reported EBITDA → Normalized EBITDA → Adjusted EBITDA**.

## What to Build

### 1. Add Normalization Data to Each Month's `makeMonth()` Block

Add a new `normalization` parameter to each month. Data structure:

```javascript
normalization: {
  sevPTO: 2399,        // severance + PTO payouts (non-recurring)
  oneTime: 0,          // other one-time items above the line
  items: []            // description strings for tooltip/detail
}
```

**2025 monthly values (from financials.md):**

| Month | sevPTO | oneTime | items |
|-------|--------|---------|-------|
| Jan 25 | 2399 | 0 | ['PTO: Blunk $1,255, Upton $1,144'] |
| Feb 25 | 26911 | 0 | ['PTO: Jump $19,742, Barcelos $2,112, Briney $1,915, K.Johnson $3,142'] |
| Mar 25 | 11812 | 33835 | ['Sev/PTO: Freeburg, Finkton, Williams, Wise', 'Repairs & Maint $19,286*', 'Misc/Other $14,549*'] |
| Apr 25 | 0 | 0 | [] |
| May 25 | 5676 | 7125 | ['Sev/PTO: Pontes $5,676', 'Recruiting $7,125'] |
| Jun 25 | 7807 | 0 | ['Sev/PTO: Brockett $7,807'] |
| Jul 25 | 4647 | 11250 | ['Sev/PTO: Harvey $4,647', 'Outside consulting $11,250'] |
| Aug 25 | 14559 | 26026 | ['Sev/PTO: Harsch, Moore, Kuphal $14,559', 'Prof Services spike $26,026*'] |
| Sep 25 | 35478 | 15111 | ['Sev/PTO: Vicencio, Baker, Brennan, Huizar $35,478', 'Prof Services $15,111*'] |
| Oct 25 | 572 | 7672 | ['PTO: Blair $572', 'Recruiting $7,672'] |
| Nov 25 | 0 | 7910 | ['Travel spike $7,910*'] |
| Dec 25 | 6099 | 438978 | ['PTO: Schilber $6,099', 'SW&Maint catch-up $136K', 'Legal reserves $151K', 'DL reclass $166K', 'IUS credit -$91K', 'Office Supplies $50K', 'Lease catch-up $8K', 'Depreciation $19K'] |

Items with * = assumptions pending Citrin verification.

**Q1 2026 values (estimates, pending Citrin):**

| Month | sevPTO | oneTime | items |
|-------|--------|---------|-------|
| Jan 26 | 577 | 14743 | ['PTO: Wallace ~$577*', 'Bank fees $14,743*'] |
| Feb 26 | 1431 | 0 | ['PTO: Morozov ~$508*, Hughes ~$923*'] |
| Mar 26 | 19162 | 0 | ['DinYero Johnson $16,242*', 'Unknown term ~$2,920*'] |

### 2. Add Normalized EBITDA Row to P&L View

In the P&L table, after the current EBITDA row, add:

```
EBITDA (Reported)          $xxx,xxx    ← existing row
  Sev/PTO Addback          $xx,xxx    ← new, indented, lighter color
  One-Time Adjustments      $xx,xxx    ← new, indented, lighter color  
Normalized EBITDA          $xxx,xxx    ← new row, bold, same weight as EBITDA
  Owner Addbacks            $xx,xxx    ← existing addback total
Adjusted EBITDA            $xxx,xxx    ← existing row
```

Color logic: These are cost addbacks, so positive = green (adding back costs improves EBITDA).

### 3. Double-Counting Guard

**CRITICAL**: Severance/PTO already appears in the existing addback schedule. When calculating Adjusted EBITDA:

```
Adjusted EBITDA = Normalized EBITDA + (Owner Addbacks - Severance/PTO already in normalization)
```

The existing addback schedule has a "Severance & PTO" line item. If that's the same data, subtract it from addbacks when adding to Normalized EBITDA to avoid double-counting. The current addback schedule total of $1,297,247 likely already includes some or all of the $115,960.

### 4. Dashboard KPI Card

Consider adding a "Normalized EBITDA" KPI card or showing it as a secondary number on the existing EBITDA card. Format: `$X.XXM (XX.X%)` margin.

## What NOT to Do
- Don't change the existing addback schedule or its total
- Don't remove the current Adjusted EBITDA calculation
- Items marked * are assumptions — display them but note they're unverified
- Don't present the normalization as final PE numbers — this is a working tool

## Reference
All data is in `.claude/memory/financials.md` under "## EBITDA Normalization Framework"
