# PE Investment Memorandum

## Document Info
- File: `dashboards/Datapath - Confidential Investment Memorandum.docx`
- Source: User-authored narrative uploaded April 2026
- 339 paragraphs, validated
- The user's uploaded version is the MASTER — never regenerate from pe-book.js

## Sections
1. Executive Summary
2. Company Overview & Transformation
   - 2.1 includes Scott Gordon bio
   - 2.4 AI section: "Client-Facing AI & Real-Time Transparency" + "Internal AI & Operational Automation"
3. Financial Performance
   - NO Gross Profit Analysis section (was in pe-book.js, not in user's version)
4. EBITDA Bridge & Addbacks (19 items, $1,297K total, Adjusted EBITDA $2.9M, margin 16.3%)
5. Customer Retention
6. Team & Operations — NO separate executive leadership paragraph (Scott bio is in 2.1 only)
7. Growth Strategy
   - 7.1 AI Suite, 7.2 Organic Growth, 7.4 Acquisition-Enabled Growth, 7.5 Margin Improvement
   - NOTE: no 7.3 in document (intentional skip)
8. Supporting Materials

## Narrative Choices
- Company age: "20-year" (user's choice for PE narrative — differs from "25 years" used elsewhere)
- Co-CEO sentence: "The Company is led by Co-CEOs James Bates and David Darmstandler supported by a seasoned executive team."
- NO Datapath Vision Framework section (was in pe-book.js version)

## Editing Workflow (XML unpack/edit/repack)
```bash
# 1. Copy original to working directory
cp "dashboards/Datapath - Confidential Investment Memorandum.docx" /sessions/dazzling-awesome-bell/original.docx

# 2. Unpack
python3 /sessions/dazzling-awesome-bell/mnt/.claude/skills/docx/scripts/office/unpack.py \
  /sessions/dazzling-awesome-bell/original.docx \
  /sessions/dazzling-awesome-bell/unpacked/

# 3. Edit document.xml directly with Edit tool
# Target: /sessions/dazzling-awesome-bell/unpacked/word/document.xml

# 4. Repack
python3 /sessions/dazzling-awesome-bell/mnt/.claude/skills/docx/scripts/office/pack.py \
  /sessions/dazzling-awesome-bell/unpacked/ \
  /sessions/dazzling-awesome-bell/output.docx \
  --original /sessions/dazzling-awesome-bell/original.docx

# 5. Validate
python3 /sessions/dazzling-awesome-bell/mnt/.claude/skills/docx/scripts/office/validate.py \
  /sessions/dazzling-awesome-bell/output.docx

# 6. Copy to dashboards
cp /sessions/dazzling-awesome-bell/output.docx \
  "dashboards/Datapath - Confidential Investment Memorandum.docx"
```

## Legacy Generator (DEPRECATED)
- File: `/sessions/dazzling-awesome-bell/pe-book.js`
- Run: `NODE_PATH=/usr/local/lib/node_modules_global/lib/node_modules node pe-book.js`
- Contains correct addback data array (19 items) but DO NOT use to regenerate
- Key constants: NAVY="1B3A5C", ACCENT="2E75B6", rev2025=17737413, ebitda2025=1595160

## P&L to EBITDA Spreadsheet
- File: `dashboards/Datapath 2025 P&L to EBITDA.xlsx`
- 12 months of 2025 (Jan-Dec) + addback section below EBITDA
- Revenue: $17,737,413 | COGS: $11,092,086 | GP: $6,645,327 | OpEx: $3,293,096 | EBITDA: $1,595,160
- Built with openpyxl, navy/blue theme, frozen panes at B6, landscape orientation
- Addback section: rows 80-104, 19 items
- Monthly Total Addbacks: `=SUM(B84:B100)` (excludes Facilities rows 82-83)
- Annual Total Addbacks: `=SUM(N82:N100)` (includes Facilities)
- Adjusted EBITDA: $2,892,407

## Datapath Vision Framework (reference, not in PE book)
- Purpose: Connect every client to the AI economy
- North Star (2030): Every client runs on Datapath AI Suite
- 4 pillars: AI Readiness & Security, Intelligent Workplace, Insight & Automation, AI Governance & Education
- Goal: Most trusted AI-driven infrastructure partner in the U.S.

## Partnership Expense Detail (Feb 2025 example)
- David Darmstandler: Auto $3K, CPA $6.6K, Meals $2.9K = $12.5K
- James Bates: Auto $4.9K, GoBundance $20.9K, Meals $8.1K, CPA $2.8K = $36.7K
- Total: $49.2K
