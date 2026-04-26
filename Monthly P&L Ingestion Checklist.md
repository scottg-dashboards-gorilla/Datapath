# Monthly P&L Ingestion Checklist

**When:** Each month after receiving the P&L from Citrin Cooperman
**Where:** Start in Claude Cowork, then hand off to Claude Code

---

## Pre-Flight
- [ ] Confirm whether P&L is **preliminary** or **final** (Citrin sends prelim first, final ~1 week later)
- [ ] Upload the P&L Excel file to Cowork

## Phase 1: Cowork — Ingest & Analyze
- [ ] Run the **monthly-pl-ingestion** scheduled task (or do manually):
  - [ ] Extract Revenue, COGS, GP, Sales Labor, OpEx, EBITDA from P&L Condensed
  - [ ] Extract GL-level detail from P&L Detail by Account
  - [ ] Compare to prior month — flag swings >15% or >$5K
  - [ ] Compare to preliminary (if this is the final version)

## Phase 2: Cowork — Gusto Pulls (targeted, not blanket)
- [ ] **Terminations**: Any new departures? Pull term dates, final payroll gross, PTO payout amounts
- [ ] **Off-cycle payrolls**: Check for bonus/commission runs that explain P&L swings
- [ ] **New hires**: Note department, rate, start date
- [ ] **Only pull what the P&L anomalies require** — don't dump the full roster

## Phase 3: Cowork — Update Knowledge Base
- [ ] Append final month numbers to `.claude/memory/financials.md`
- [ ] Add prelim→final changes (if applicable)
- [ ] Add new Citrin questions / anomalies
- [ ] Add complete dashboard data mapping (all revenue/COGS/labor/OpEx values)
- [ ] Add termination/PTO addback data from Gusto
- [ ] Update headcount if changed

## Phase 4: Sync to Claude Code
- [ ] Save updated `financials.md` to Mac repo (use `present_files` or manual copy)
- [ ] Verify Claude Code can read `.claude/memory/financials.md`
- [ ] Tell Claude Code to:
  - [ ] Add new month's data block to dashboard (makeMonth + DATA array)
  - [ ] Fix any prior month corrections identified
  - [ ] Update EBITDA spreadsheet addbacks if new items found
  - [ ] Push to GitHub

## Phase 5: Verification
- [ ] Check dashboard loads with new month visible
- [ ] Verify EBITDA totals match P&L
- [ ] Confirm GP% by department looks reasonable vs. benchmarks
- [ ] Review any Citrin follow-up questions before next meeting

## Citrin Follow-Up (if needed)
- [ ] Draft questions for unresolved anomalies
- [ ] Send follow-up email
- [ ] Update knowledge base when answers received

---

## Key Reminders
- **Preliminary vs. Final**: Dashboard should use final numbers. If only prelim available, note it and correct when final arrives.
- **R&D classification**: Dashboard puts R&D in COGS, Citrin in OpEx. Same EBITDA — not a discrepancy.
- **Dashboard COGS grouping**: Backup/spam/domain/voice go INTO managed_services COGS on dashboard.
- **Never break out Scott Gordon's comp individually.**
- **Gusto ≠ Citrin dollar-for-dollar**: Citrin posts from Intacct JEs, not directly from Gusto. Timing differences are normal.
