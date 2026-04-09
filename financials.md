# Financial Data & EBITDA Addback Schedule

## Revenue Departments (8)
1. Managed Services (MS) — ~$593K/mo (includes Service Delivery + Account Mgmt)
2. Security (MDR/SOC) — ~$150K/mo
3. Cloud & Hosting — ~$66K/mo
4. Projects / Prof. Services — ~$21K/mo (internal project team only)
5. Subcontracted Services — ~$4K/mo (outsourced one-time work)
6. HW/SW Resale — ~$423K/mo (lumpy, low margin)
7. AI Consulting — $0 currently (future AI consulting revenue)
8. R&D — cost center, $0 revenue, ~$75K/mo COGS (new in 2026)

## COGS Structure
- MS COGS = Tech Stack (tools/software ~$43K) + Labor (Service + AM combined)
- Security COGS = vendor costs + security team labor
- PS COGS = project engineer labor (75% of their time)
- Cloud COGS = vendor costs + Peter Annabel 25%
- HW/SW COGS = product costs + procurement team labor
- R&D COGS = payroll ($63,968) + taxes ($5,914) + benefits ($1,449) + Haskell 1099 ($4,000) = $75,331
- Subcontracted COGS = subcontractor costs only (no labor allocation)

## MSP Benchmarks (GP Targets)
MS: 72% | PS: 55% | Cloud: 70% | Security: 65% | HW/SW: 22% | vCIO: 75% | Subcontracted: 40%

## Labor Allocation Rules

### Legacy Model (2025 and earlier)
- Direct Labor → 100% into Managed Services COGS
- 25% of Project Labor → into Managed Services COGS
- PS Head ($11K/mo) → into Security COGS
- Remaining Project Labor → into Professional Services COGS
- Sales Labor → below the line (not in COGS)
- Admin Payroll → OpEx

### New Model (Jan 2026+, roster-based)
- makeMonth() supports `labor.byDept` parameter for per-department allocation
- Total direct+project labor redistributed proportionally by gross wages across COGS depts
- Each employee assigned to specific department(s) with split percentages
- Sales Labor → separate P&L line (opex, not COGS)
- R&D → separate COGS line
- Jan 26 byDept: MS $194K, PS $34K, Security $22K, Cloud $3K, HW/SW $10K, R&D $24K
- Feb 26 byDept: MS $134K, PS $25K, Security $16K, Cloud $2K, HW/SW $7K, R&D $0

## G&A Sub-Buckets
| Sub-Bucket | People | Monthly Total |
|-----------|--------|--------------|
| Sr Leadership | David D., Scott G., Dan S. 50%, Stephen W. 50% | $45,071 |
| Central Services | Marshall Rownd, Devin Peterson | $16,389 |
| Finance | Melinda Hennington, Jaquelene Dizon | $8,896 |
| Marketing | Myca Villanueva | $1,985 |
| HR | Nahili Bekele | $1,600 |
| Admin | Colin Bates, Hudson Darmstandler | $3,520 |

## 2025 EBITDA Addback Schedule (19 items, $1,297,247 total)
Updated April 2026 from "Datapath 2025 P&L With Add-Backsv2.xlsx"

| # | Item | Amount | Type | Monthly Allocation |
|---|------|--------|------|-------------------|
| 1 | Facilities Modesto (HQ) | $188,400 | Normalize | $15,700/mo x 12 |
| 2 | Facilities Ohio | $20,000 | Normalize | $4,000/mo Aug-Dec |
| 3 | COO Consulting Premium (50%) | $153,000 | Normalize | $12,750/mo x 12 |
| 4 | Website Development | $10,888 | One-Time | Apr $6K + Dec $4,888 |
| 5 | Marketing GTM Consulting | $42,000 | One-Time | $6,000/mo Jun-Dec |
| 6 | Family Member Comp | $42,240 | Normalize | $3,520/mo x 12 |
| 7 | Severance & PTO Payouts | $115,960 | One-Time | Weighted heavier Jan-Mar |
| 8 | Year-End Legal Reserves (PAGA) | $231,000 | One-Time | Dec only |
| 9 | R&D Tax Credit Study 2022-2024 | $70,940 | One-Time | Dec only |
| 10 | Citrin Cooperman Start Assessment | $25,000 | One-Time | Jan only |
| 11 | Phoenix Strategy Group | $17,000 | One-Time | Jan-Feb ($8,500/mo) |
| 12 | Roger Gentry Consulting | $48,000 | One-Time | Jan-Jun ($8,000/mo) |
| 13 | Podcast Studio Build-Out | $30,000 | One-Time | Mar-Apr ($15,000/mo) |
| 14 | AP Logic (SharePoint) | $10,000 | One-Time | Feb only |
| 15 | Ohio Office Flood Repairs | $20,000 | One-Time | Mar only |
| 16 | TPC Contract Negotiation (Legal) | $20,000 | One-Time | Aug only |
| 17 | 11:11 Systems Contract Buyout (PIA) | $55,566 | One-Time | $6,174/mo Jan-Sep |
| 18 | Additional Legal Fees (Litigation) | $62,000 | One-Time | ~$5,167/mo spread |
| 19 | Balance Sheet Cleanup Adj for Insurance | $135,253 | Non-Cash | Dec only |

Composition: One-Time $758K (58.5%), Normalize $404K (31.1%), Non-Cash $135K (10.4%)
Still TBD: DeHart HVAC, David Bates fees, Inforcer tool costs, R&D costs

### Changes from Prior Version
- REMOVED: McEachron Ministries ($12K), Charitable Contributions ($37.4K)
- ADDED: Facilities Modesto ($188.4K), Facilities Ohio ($20K), Website Dev ($10.9K), Marketing GTM ($42K)
- CHANGED: R&D Tax Credit $27.8K→$70.9K; 11:11 Systems $22K→$55.6K; Year-End Legal $231K re-added

## Dec 2025 Year-End Adjustments
- 23 journal entries moved net income from -$189K to -$6K
- Other Expense $601K credit | Prof Services $231K (legal reserves)
- Direct Labor $161K reclass | Insurance $135K (prepaid cleanup)
- D&A $316K (depreciation $102K + amortization $142K + finance lease $72K)
- Software capitalization $91K | Cloud revenue -$60K (HHHC correction)
- Updated P&L received April 2026 — EBITDA improved ~$99K ($1,496K→$1,595K)

## Valuation & Exit Strategy
- Target multiple: 8-11x adjusted EBITDA (strategic acquisition)
- Target purchase price: $30M–$35M
- Every $100K in add-backs = $800K–$1.1M in enterprise value

## Citrin Cooperman P&L Import Format
- Sheets: "P&L Condensed" (summary) and "P&L Detail by Account" (account-level)
- "All Locations" column: scan row 5 for "All Location", take FIRST match
- Revenue accounts: 40000→MS, 40100/42100→Security, 41360/41365/41300→Cloud, 41200/41250/41450/42000/42200/42300/44000→HW/SW, 41500→PS, 41700→Subcontracted
- COGS: "Managed IT Service Costs"→MS, "Cost of Monthly Recurring Cloud"→Cloud, "Cost of Monthly Recurring Resale"+"Product Sales Cost"→HW/SW, "Subcontractor Service Costs"→Subcontracted
- Labor: "Direct Labor Costs"→direct, "IT Project Labor"→project, "Sales Labor Costs"→salesLab
- OpEx: payroll, facilities, operating, marketing, ga, taxIns
- Other Income & Tax: from "P&L Detail by Account" sheet

## Annual Totals (Reference)
- 2023: Rev $27.7M / Net $3.4M
- 2024: Rev $22.1M / Net $1.1M
- 2025: Rev $17.8M / Net $1.3M
