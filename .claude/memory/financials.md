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

## Capitalized Payroll / Internal-Use Software (IUS)
Source: "DP IUS 3.31.26 (2).xlsx" from Citrin Cooperman (Peter Annabel quarterly review)
GL Account: 50661 (Payroll Expenses - Capitalized), reduces Direct Labor
Amortization GL: 19003, 3-year useful life

### People Capitalized
- **Peter Annabel** (Dir of Cloud Services) — $9,233/mo gross, LOE 30-95% depending on project
- **Haskell Macarcig** (1099 contractor) — $4,000/mo flat, 100% LOE, coded to GL 61900 (R&D)
- **Tim Culcasi** (QA) — $18,444/mo gross, only 5% allocation ($922 total Q1 2026)
- **Roger Gentry** (former contractor) — $40K total, 20% LOE haircut (work was rebuilt after he left Jun 2025)

### Capitalized Projects
| Project | Period | Extended Cost | Status |
|---------|--------|-------------|--------|
| Vision Dashboard (core build) | Jun-Nov 2025 | $67,463 | Live 11/1/25, amortizing |
| Feature 1: Addt'l Data Sources | Nov-Dec 2025 | $2,318 | Live 12/31/25, amortizing |
| Feature 2: Trend Lines | Nov-Dec 2025 | $2,318 | Still dev/beta, ON HOLD |
| Feature 3: LLM Layer/UI/UX | Nov 2025-Feb 2026 | $43,697 | Live 3/2/26, amortizing from Mar |
| Enhancement (logging ingestion) | Feb 2026 | $1,385 | Live 2/18/26 |
| Vision Testing Tool (LLM regression) | Mar 2026 start | $1,806 | Still in dev |

### Totals
- Through 12/31/2025: $90,648
- Q1 2026 (posted in March): $28,341
- Grand total: $118,989
- Monthly amortization as of March 2026: $3,191/mo (ramping as features go live)
- Quarterly catch-up: Citrin meets with Peter Annabel once/quarter; all Q1 posted in March
- **Recommendation accepted**: normalize monthly with quarterly true-up going forward

### Key Notes
- Haskell was previously coded to GL 61100 (Marketing) through Jan 2026; reclassed to GL 61900 (R&D) starting Feb 2026
- Roger Gentry's $40K was haircut to $8K (20% LOE) because his work was rebuilt by Peter after Roger left
- Tim Culcasi QA time is minimal (5%) — mostly testing, not core development
- Feature 2 (Trend Lines) is on hold — "very little dev since December, still in Beta testing"
- This is separate from R&D COGS ($42-75K/mo): capitalized payroll pulls costs OFF the P&L, R&D COGS is a reclassification WITHIN the P&L
- No overlap confirmed between people: Peter/Haskell are in IUS capitalization; Tim Stewart/Jacob/James are in R&D COGS

## March 2026 Preliminary P&L — Citrin Q&A Status

### Answered (5 of 15 anomalies)
1. **MS COGS Tech Stack -34%**: SVH/Trumark overbilling correction ($6,240). True run rate ~$30-31K. Resolved.
2. **Capitalized Payroll -$28,341**: Vision dashboard project, quarterly catch-up. See IUS section above. Action: normalize monthly.
3. **OpEx jumped +$43K**: Pia Trade Co AI desk licenses ($29K) posted 2025, reversed Feb 2026. Resolved.
4. **Other Taxes $18,900**: Franchise tax payments (CA $14.4K + OH $4.5K). Action: normalize monthly ~$6,300.
5. **Project Services +181%**: Actual invoiced work, not catch-up. Good news.

### Answered Post-Initial (from Citrin emails April 16)
7. **Officer's Comp negative in Feb (-$10,769)** — Citrin reclassed James's YTD payroll from Officer's Comp into the new R&D bucket, plus moved several other people's Jan payroll retroactively. Same journal entry that caused the Feb R&D $75,331 catch-up. Resolved.
8. **Professional Services $76K/mo** — Citrin provided full vendor breakdown (3-mo avg). Recurring: Citrin $28K, CLevel $25.5K, Call & Jensen $5.6K, Sensiba $3.8K, CyberGuard $2.4K, Avalara $1.9K, Lucas/Horsfall $1.9K = ~$69K recurring. One-time/ad-hoc: Silvermine AI $3.75K, Rank Investigations $900, Katherine Boyd $738, LegalZoom $499, NexNow $458, Dean Dorton $400, Osborne $353, INC Installs $314, InCorp $129 = ~$7.5K. Total $76,509.
9. **Insurance $450→$10,335** — Feb was low due to $3.5K refund. March has $7,020 Travelers charge (April recurring from Sage posted early). True run rate ~$7-7.5K/mo. Will monitor, no normalization needed.
11. **OH Interest Expense tripled** — Posting error: interest/principal not reconciled to BS. Now fixed. One of two FFB loans paid off in Jan. Corrected Q1: Jan $16.8K, Feb $15.8K, Mar $15.6K (declining trend). Resolved.

### Answered by Kasey (Payroll Support Q1 2026, April 2026)
6. **Direct Labor Wages +$30K in CA** — RESOLVED. January had 3 payroll runs vs 2 in Feb/Mar. Jan DL gross was $209,406; normalized to 2 payrolls = ~$139,604, vs Feb $146,742 and Mar $135,736. March DL dropped $11K from Feb due to dismissals: Hughes ($3.2K), Morozov ($2.6K), Doyle ($3.2K reduction), Chavez Soto maternity ($3.1K). No anomaly — payroll timing + terminations.
7b. **Officer's Comp** — RESOLVED. Kasey confirmed: R&D reclassification journal entry moved James Bates + others from Officer's Comp into R&D. Feb showed negative Officer's Comp as retroactive YTD correction. Gusto JE template was also incorrect (pushing to wrong Intacct accounts), corrected for April forward.
10. **R&D dropped $75K→$43K** — RESOLVED. Feb $75K included retro YTD catch-up (Jan+Feb R&D all posted in Feb). Go-forward R&D payroll run rate = ~$26,505/mo (Annabel $9,233 + J.Bates $6,550 + T.Stewart $6,722 + Berlin $4,000). Feb was inflated by Tim Stewart $5K bonus + retroactive Jan posting. Citrin P&L shows $42,943 for Mar which also includes taxes/benefits on top of gross wages.

### Q1 2026 Payroll Key Facts (from DP Payroll Support file)
- **3-payroll January**: Jan had 3 payroll runs ($481,931 total) vs 2 in Feb ($335,778) and Mar ($343,764). Normalize Jan by ×2/3 = ~$321,287.
- **Personnel changes**: Hughes dismissed Feb, Morozov dismissed Feb, Wallace dismissed Jan, Chavez Soto maternity leave Mar
- **DinYero Johnson sales spike**: $14,618 (Feb) → $31,283 (Mar), +$16,665 — biggest single swing in March. Drove sales labor from $80K→$101K.
- **Tim Stewart $5K bonus**: Approved by James, paid Feb. Normal R&D rate ~$6,722/mo.
- **Jacob Berlin**: New R&D hire, started Feb ($1K), ramped to $4K/mo in Mar.
- **Gusto JE fix**: Template was incorrectly pushing payroll to Intacct accounts. Corrected for April forward.
- **Department totals (Mar)**: Admin $29,135 | DL $135,736 | Officer $8,299 | Project $42,703 | R&D $26,505 | Sales $101,385

### Not Asked (lower priority, user aware)
- Product Sales +$123K (user aware of dynamics)
- Subcontracted tripled $4K→$15K (user aware)
- Sales Commissions zeroed out

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
