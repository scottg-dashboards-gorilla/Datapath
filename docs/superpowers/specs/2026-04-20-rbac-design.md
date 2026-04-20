# RBAC Dashboard Expansion — Design Spec
**Date:** 2026-04-20  
**Status:** Approved by user  
**Scope:** Expand `monthly financial analysis.html` to support role-based access for department heads, a full admin UI, and associated security hardening.

---

## Overview

The dashboard currently has a single SHA-256 password gate giving full access to all data. This spec adds a two-tier role system:

- **Admins** (Scott, David, James) — see everything they see today, plus an Admin tab for user management
- **Dept Heads** (5 users) — scoped view of their assigned departments only, with a purpose-built shell

Everything stays in a single HTML file. No server, no backend, no cookies.

---

## 1. User Configuration

A `const USERS = [...]` block near the top of the file defines all accounts:

```js
const USERS = [
  {
    id: 'swalski',
    name: 'Stephen Walski',
    username: 'swalski',
    passwordHash: '<sha256(salt+password)>',
    salt: '<random 16-char string>',
    role: 'dept_head',
    departments: ['professional_services', 'security'],
    views: ['overview', 'pl', 'gross_profit', 'labor'],
  },
  // ... other users
  {
    id: 'scott',
    name: 'Scott Gordon',
    username: 'scott',
    passwordHash: '...',
    salt: '...',
    role: 'admin',
    departments: [],   // admins see all
    views: [],         // admins see all
  },
];
```

**Password hashing:** SHA-256 over `salt + password` (concatenated). Each user gets a unique random salt stored in their record. Prevents rainbow table attacks on the stored hashes.

**Repo privacy note:** The HTML file (with hashes + salts) auto-pushes to GitHub Pages. Ensure the GitHub repo is set to **private** before dept head accounts are active.

### Initial access matrix

| User | Role | Departments | Views |
|------|------|-------------|-------|
| Scott Gordon | admin | all | all |
| David Darmstandler | admin | all | all |
| James Bates | admin | all | all |
| Stephen Walski | dept_head | PS, Security | Overview, P&L, Gross Profit, Labor |
| Brandon H. | dept_head | Managed Services | Overview, P&L, Gross Profit, Labor |
| Tim C. | dept_head | AI / R&D | Overview, P&L, Gross Profit, Labor |
| Nathan R. | dept_head | AM, Inside Sales | Overview, P&L, Gross Profit, Labor, Sales |
| Dan S. | dept_head | Marketing, Sales | Overview, P&L, Gross Profit, Labor, Sales, Marketing |

---

## 2. Authentication & Session Management

### Login screen
Light/clean style: white card on light gray background, blue "DATAPATH" wordmark, username + password fields, "Sign In" button. Matches existing dashboard aesthetic.

### Login flow
1. User enters username + password
2. Lookup user record by username
3. Compute `SHA-256(user.salt + enteredPassword)`, compare to `user.passwordHash`
4. On match: write session to storage, render appropriate shell
5. On failure: increment attempt counter (stored in `sessionStorage` keyed by username)

### Brute-force lockout
- After **5 failed attempts** for a given username: lock that username for **15 minutes**
- Lockout state stored in `sessionStorage` (clears on browser close)
- Show countdown message: "Too many attempts. Try again in X minutes."

### Session storage
- **Dept heads:** `sessionStorage` — session clears when tab is closed
- **Admins:** `localStorage` — persists across browser restarts (matches existing behavior)
- Both: **1-hour inactivity timeout** (reset on any interaction)
- Session token stores: `{ userId, role, loginTime, lastActive }`

### Audit log
- Every login (success + failure), logout, and tab navigation is appended to an in-memory audit array
- Also written to `localStorage` under key `dp_audit_log` (last 500 events, rolling)
- Format: `{ ts, userId, action, detail }`
- Viewable from the Admin panel (read-only table)
- Useful for demonstrating controlled access during PE due diligence

---

## 3. Admin Experience

Admins see the existing dashboard exactly as today. One addition: an **Admin** tab appears in the navigation bar (never visible to dept heads).

### Admin tab — User Management panel

**Layout:** Master/detail — user list sidebar on the left, edit panel on the right.

**Left sidebar:**
- Lists all dept head accounts (admins not editable here — protect against self-lockout)
- Click a name → their settings load in the right panel
- "+ Add User" button at the bottom

**Right panel — per user:**
- Username (editable text field)
- Password reset (type new password → hashed on save; leave blank to keep current)
- Department toggles — pill buttons, click to grant/revoke (blue = granted, gray = revoked)
- View toggles — same pill pattern for the 6 available views
- "Remove User" button (red, requires confirmation)
- "Save Changes" button — writes updated `USERS` block back into the page's own `<script>` tag using a string replace, then rehashes any changed passwords

**Available views (toggle labels):**
1. Overview
2. P&L
3. Gross Profit
4. Labor
5. Sales
6. Marketing

### Admin tab — Audit Log panel

Sub-tab within Admin. Read-only table: Timestamp / User / Action / Detail. Filterable by user and date range.

### Admin commentary (per month, per department)

Admins can add free-text notes to any month + department combination. Stored in a `const COMMENTARY = {}` block in the file, keyed by `'Mon YY:dept_key'`.

Example:
```js
const COMMENTARY = {
  'Mar 26:professional_services': 'Revenue spiked due to TPC project — not indicative of run rate.',
  'Mar 26:security': 'Vendor cost increase from new MDR tooling; one-time onboarding fee included.',
};
```

Commentary is editable inline in the Admin panel (text area next to the month/dept selector). Dept heads see it as an amber callout box at the top of their KPI view when a note exists.

---

## 4. Department Head Experience

### Shell
- **Header bar** (blue, `#1e40af`): "DATAPATH" wordmark on left, role badge in the middle (e.g., "Prof. Services + Security"), username + sign-out on right
- **Dept switcher pills** (below header, white bar): "All Depts" + one pill per assigned department. Active pill is blue. Switching filters all KPIs and the breakdown table.
- **Month selector**: visible, same styling as admin month selector. Shows PRELIMINARY badge on preliminary months.
- **"Data through" indicator**: small gray text below month selector — "Data through Mar 2026 · PRELIMINARY" or "Data through Mar 2026 · Final"

### KPI cards
Three cards in a row (or stacked on mobile):
- **Revenue** — blue left border, MoM delta below
- **Gross Profit** — green left border, GP% below
- **Labor** — amber left border, headcount below (if applicable)

Each card includes a **6-month sparkline** (tiny inline SVG line chart) below the main number showing that metric's trend over the last 6 months. Scoped to selected dept(s).

No company-wide revenue, no EBITDA, no other departments' data — ever.

### Per-dept breakdown table
When "All Depts" is selected and user has 2+ departments: a table below the KPI cards showing each dept as a row — Department / Revenue / GP / GP%.

### Available views (dept heads only see their granted views)
Tabs appear only for views the user has access to. Tab order: Overview → P&L → Gross Profit → Labor → Sales → Marketing.

### Admin commentary display
If `COMMENTARY['Mon YY:dept_key']` exists for the current month + dept, an amber callout box appears above the KPI cards:
> **Note from leadership:** Revenue spiked due to TPC project — not indicative of run rate.

### Dept-scoped Claude advisor
The existing Claude API advisor (claude-sonnet-4-20250514) appears for dept heads with a modified system prompt that:
- Restricts context to only their departments' data
- Explicitly instructs the model not to reveal company-wide revenue, EBITDA, or other departments' numbers
- Frames responses as "your department" context
- Uses the same API key flow (stored in localStorage)

### Mobile layout
Dept head shell is responsive:
- Header: stacks to two lines on narrow screens
- Dept pills: horizontal scroll if overflow
- KPI cards: single column on mobile
- Breakdown table: horizontally scrollable

### PDF export
A "Download Report" button in the dept head header generates a print-formatted view of the current month's KPIs, breakdown table, and commentary (if any), then triggers `window.print()` with a print-specific stylesheet. No external libraries needed.

---

## 5. Data Scoping

All financial data remains in the existing `DATA` array. Dept head views filter it at render time:

- Revenue, COGS, GP: sum only the user's assigned `departments` keys
- Labor: use `byDept` values for their departments only (2026+); for earlier months use the legacy proportional estimate
- OpEx, EBITDA: **never shown** to dept heads
- Sparkline history: same filter applied across the last 6 months of `DATA`

No data is removed from the file — scoping is purely in the rendering layer.

---

## 6. Budget vs. Actual (future-ready)

Add an optional `budget` key to each month object, keyed by department:

```js
MAR26.budget = {
  professional_services: { revenue: 50000, gp: 45000 },
  security: { revenue: 150000, gp: 90000 },
};
```

If `budget` exists for a month, dept head KPI cards show a "vs. Budget" delta below the MoM delta. If absent, the delta row is hidden. No UI for budget entry in this release — data is hand-coded by admins. The data model is ready; the feature ships when budget data exists.

---

## 7. Implementation Notes

- The `USERS` and `COMMENTARY` blocks must be clearly delimited with sentinel comments so the admin save logic can find and replace them via string manipulation:
  ```js
  // @@USERS_START@@
  const USERS = [...];
  // @@USERS_END@@
  ```
- Admin save uses `document.documentElement.outerHTML` replacement + a `Blob` download (since we can't write to disk from the browser). User downloads the updated HTML and replaces the file manually — OR we use the existing auto-push flow (admin saves → downloads → drops into the repo folder → push picks it up).
- Alternatively: if the dashboard is opened as a local file (`file://`), the File System Access API (`showSaveFilePicker`) can write back directly. Progressive enhancement: try FSA first, fall back to download.
- All existing admin functionality (full P&L views, Claude advisor, addback schedule) is unchanged.

---

## 8. Out of Scope

- Password recovery / forgot password flow (admins reset manually)
- Multi-factor authentication
- Per-month access restrictions (all months visible to all dept heads)
- Email notifications for new data (no server)
- Real-time collaboration or shared state
