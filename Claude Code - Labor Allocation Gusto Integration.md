# Claude Code: Gusto-Powered Editable Labor Allocation

## Overview
Replace the hardcoded `LABOR_DATA` object in `renderLaborAllocation()` (line ~4054 of the dashboard) with data sourced from Gusto, while preserving the existing department assignments and allocation percentages as editable overrides. The user wants to be able to change any field (department, allocation %, title) inline and have those changes save to the knowledge base.

## Current State
- `LABOR_DATA` (line 4054-4181) is a static JS object with 46 W-2 employees + ~10 contractors
- Each person has: `{n: name, t: title, type: 'W-2'|'BruntWork'|etc, g: grossMonthly, p: 'allocation%'}`
- People are grouped into departments: 6 COGS depts, R&D, Sales, 6 G&A sub-depts
- Some people appear in multiple departments with split allocations (e.g., Keith Newson 25% MS + 75% PS, Stephen Walski 50% Security + 50% G&A Sr Leadership)

## What Gusto Provides (46 active employees)
Gusto departments DO NOT match the dashboard departments. Here's the mapping:

| Gusto Department | Dashboard Department(s) |
|------------------|------------------------|
| West Coast - Service Delivery | Managed IT — Service Delivery (100%) |
| East Coast - Service Desk | Managed IT — Service Delivery (100%) |
| East Coast - Field Engineering | Managed IT — Service Delivery (100%) |
| West Coast - Engineering | Split: 25% MS Service Delivery + 75% Projects/PS |
| West Cost-Fresno Engineering | Split: 25% MS Service Delivery + 75% Projects/PS |
| East Coast - Installation Services | Managed IT — Service Delivery (100%) |
| West Coast - Sales + Marketing | Split: varies per person (Sales, AM, G&A) |
| East Coast - Sales and Marketing | Split: varies per person (Sales, AM) |
| West Coast - Education | Split: 50% Sales + 50% Account Mgmt |
| Cyber Security | Security (MDR/SOC) (100%) |
| Cloud Services | Split: 25% Cloud COGS + 75% R&D |
| Central Services & Internal IT | G&A > Central Services (100%) |
| Admin Finance-Fresno | G&A > Finance (100%) |
| Procurement | HW/SW Resale (100%) |
| Sales Engineering | Sales (100%) |
| Owner | Split: Co-CEO Bates 100% R&D, Co-CEO Darmstandler 100% G&A Sr Leadership |

## What to Build

### 1. Department/Allocation Override Map
Create a `LABOR_OVERRIDES` object (stored in the dashboard and synced to `.claude/memory/employees.md`) that maps each employee UUID to their dashboard department assignment(s) and allocation percentage(s):

```javascript
const LABOR_OVERRIDES = {
  // UUID: { depts: [{dept: 'dashboard_dept_id', pct: 100}], titleOverride: null }
  '9186e739-...': { depts: [{dept:'managed_services_delivery', pct:25}, {dept:'professional_services', pct:75}], titleOverride: 'Engineering Team Manager' },
  // Keith Newson — Gusto says "West Coast - Engineering", dashboard splits him 25/75
};
```

Pre-populate this with the current assignments from `LABOR_DATA`. When Gusto data refreshes, names and base salaries update automatically, but department assignments and allocation splits come from the override map.

### 2. Salary Source
For W-2 employees, pull from Gusto compensations:
- Salary employees: `rate / 12` = monthly gross
- Hourly employees: `rate * 173.33` = monthly gross (assumes 40hrs/wk, 4.333 wks/mo)

For contractors (BruntWork, Microsourcing, CRDLE, Direct, 1099), keep the existing hardcoded amounts since they're not in Gusto payroll. Flag them visually as "manual entry."

### 3. Current Gusto Rates (for reference)
| Name | Gusto Rate | Type | Monthly Est. |
|------|-----------|------|-------------|
| Melinda Hennington | $90,640/yr | Salary | $7,553 |
| Devin Peterson | $93,000/yr | Salary | $7,750 |
| Marshall Rownd | $120,000/yr | Salary | $10,000 |
| Peter Annabel | $120,000/yr | Salary | $10,000 |
| Angelica Jackson | $85,000/yr | Salary | $7,083 |
| Michael Trude | $73,470/yr | Salary | $6,123 |
| Nathan Lanning | $79,762/yr | Salary | $6,647 |
| Kevin Barr | $50,000/yr | Salary | $4,167 |
| Joel Walker | $80,000/yr | Salary | $6,667 |
| Christopher Vouis | $19.00/hr | Hourly | $3,293 |
| Cole Matuszynski | $22.00/hr | Hourly | $3,813 |
| Colton Keim | $31.00/hr | Hourly | $5,373 |
| Cory Strassell | $77,155/yr | Salary | $6,430 |
| Kerr Conkle | $85,000/yr | Salary | $7,083 |
| Matthew Brinovec | $79,200/yr | Salary | $6,600 |
| Nathaniel Dupray | $25.00/hr | Hourly | $4,333 |
| Sam Dean | $32.00/hr | Hourly | $5,547 |
| David Darmstandler | $65,000/yr | Salary | $5,417 |
| James Bates | $65,000/yr | Salary | $5,417 |
| David Craig | $75,000/yr | Salary | $6,250 |
| Daniel Chadwell | $107,000/yr | Salary | $8,917 |
| Ricky Maestas | $83,554/yr | Salary | $6,963 |
| James Bruce | $91,875/yr | Salary | $7,656 |
| Keith Newson | $115,000/yr | Salary | $9,583 |
| Matthew Lampe | $97,000/yr | Salary | $8,083 |
| Stephen Walski | $143,000/yr | Salary | $11,917 |
| Daniel Sturdivant | $150,000/yr | Salary | $12,500 |
| Jay Harvey | $90,000/yr | Salary | $7,500 |
| Nathan La Fleche | $120,000/yr | Salary | $10,000 |
| Angela Contreras | $37.00/hr | Hourly | $6,413 |
| Arnold Vega | $27.00/hr | Hourly | $4,680 |
| Brandon Hulsey-Cedeno | $31.00/hr | Hourly | $5,373 |
| Brandon Johnson | $110,000/yr | Salary | $9,167 |
| Christopher Fuenty | $33.00/hr | Hourly | $5,720 |
| Colin Bates | $22.00/hr | Hourly | $3,813 |
| Cristina Chavez Soto | $25.00/hr | Hourly | $4,333 |
| Gabriel Espinoza | $37.47/hr | Hourly | $6,494 |
| Haskell Lark Macaraig | $78,000/yr | Salary | $6,500 |
| Hudson Darmstandler | $22.00/hr | Hourly | $3,813 |
| Jacob Berlin | $52,000/yr | Salary | $4,333 |
| Luis Pena | $31.00/hr | Hourly | $5,373 |
| Matthew Irick | $33.00/hr | Hourly | $5,720 |
| Oscar Contreras | $27.25/hr | Hourly | $4,723 |
| Timothy Stewart | $87,360/yr | Salary | $7,280 |
| William Doyle | $33.00/hr | Hourly | $5,720 |
| Christopher Gilmore | $108,120/yr | Salary | $9,010 |

### 4. Editable Fields in the UI
Make these fields editable inline when the user clicks on them:

- **Department assignment**: Dropdown with all dashboard departments. When changed, recalculates allocations.
- **Allocation %**: Click to edit. If person is in multiple departments, show split (e.g., "25% / 75%"). When changed, automatically adjusts the split on the other department.
- **Title**: Click to edit (overrides Gusto title for display purposes).
- **Gross/Mo**: Click to edit for contractors. For W-2, show Gusto rate (read-only) with a note that it auto-updates.
- **Type**: Read-only (from Gusto or manual designation).

### 5. Save Behavior
When any field is edited:
1. Update `LABOR_OVERRIDES` in the dashboard JS
2. Recalculate all department totals and P&L allocation percentages
3. Store the override map in `localStorage` keyed by employee UUID
4. On next knowledge base update, Cowork will pull the overrides and sync to `.claude/memory/employees.md`

### 6. Employees Not in Gusto (Contractors)
These people appear in the current `LABOR_DATA` but are NOT in Gusto. Keep them as manual entries with an "Add Manual Employee" button:

| Name | Type | Department | Gross/Mo |
|------|------|-----------|----------|
| Hoover Dimson | BruntWork | MS Service Delivery | $1,265 |
| Jenny Magcalayo | BruntWork | MS Service Delivery | $1,323 |
| Girly De La Pena | BruntWork | MS Service Delivery | $1,309 |
| Edwin De Guzman | Microsourcing | Account Mgmt | $2,074 |
| Hermella Mulugeta | CRDLE | Account Mgmt | $1,600 |
| MJ Martinez | Gusto 1099 | Account Mgmt | $2,308 |
| Julissa Martinez Duran | Gusto 1099 | Account Mgmt | $2,123 |
| Carla Humber Munguia | Direct | Account Mgmt | $2,500 |
| Genelyn Terrones | BruntWork | HW/SW Resale | $1,540 |
| Carrlyne David | BruntWork | HW/SW Resale | $1,079 |
| Jaquelene Dizon | Microsourcing | Finance | $1,922 |
| Myca Villanueva | Microsourcing | Marketing | $1,985 |
| Nahili Bekele | CRDLE | HR | $1,600 |
| Consulting & Office Space | Consulting | G&A Sr Leadership | $25,500 |

### 7. Discrepancies Between Gusto and Current Dashboard
Some salaries differ between what Gusto shows now and what's in the hardcoded `LABOR_DATA`. This is normal — the dashboard may use actual payroll amounts while Gusto shows base rate. Key differences to note:

- DinYero Johnson appears in Sales but was terminated 3/26 — REMOVE from active roster
- Angela Contreras listed as "Michelle Contreras" in dashboard — confirm correct name with Gusto (Angela)
- Henry Martinez and Mark Mercer appear in dashboard but NOT in Gusto active employees — may be terminated, verify
- Tommy Wijaya appears in dashboard but NOT in Gusto — may be terminated, verify
- Christopher Fuenty is in Gusto but NOT in current dashboard — NEW hire, needs department assignment
- Haskell Macaraig shown as "1099" in dashboard R&D but is W-2 in Gusto at $78K — update type

## What NOT to Do
- Don't remove contractors from the labor allocation — they're real labor costs even though they're not in Gusto
- Don't change the P&L allocation logic (proportional by gross wages within COGS departments)
- Don't expose individual compensation for "Consulting & Office Space" line — this is Scott Gordon's comp, always show as aggregate
- Don't auto-refresh from Gusto on every page load — use cached data, update on demand with a "Refresh from Gusto" button

## Reference
- Current `LABOR_DATA`: lines 4054-4181 of `monthly financial analysis.html`
- Employee memory: `.claude/memory/employees.md`
- Gusto connector: available via MCP tools (list_employees with include=all_compensations)
