# RBAC Dashboard Expansion — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expand `monthly financial analysis.html` from a single-password gate to a full role-based access system with dept head scoped views, an admin management UI, and security hardening.

**Architecture:** Single HTML file, all logic client-side. A `USERS` config block (sentinel-delimited) stores salted SHA-256 hashes; a `COMMENTARY` block stores per-month notes. Admins see the existing dashboard plus an Admin tab. Dept heads get a purpose-built blue-shell view scoped to their departments only.

**Tech Stack:** Vanilla JS, `crypto.subtle` (SHA-256), SVG sparklines, File System Access API (with Blob download fallback), `window.print()` for PDF. No new libraries.

**Security note on innerHTML:** The existing dashboard renders all content via `innerHTML` with data sourced from the file's own hardcoded `DATA` array — the same pattern continues here. All user-generated strings (commentary text, usernames) must be passed through an `escHtml()` helper before insertion. This helper is defined in Task 1 and must be used anywhere user-supplied text appears in HTML.

---

## File Map

All changes are in one file: `monthly financial analysis.html` (currently ~4,252 lines).

| Section | Lines (approx) | What changes |
|---------|---------------|-------------|
| Login gate HTML | 277-286 | Add username field |
| Auth JS block | 287-331 | Full replacement: multi-user, salted, lockout |
| State object | 354-361 | Add `currentUser`, `dhDept` fields |
| NAV array | 2052-2061 | Add Admin entry (conditionally rendered) |
| `renderNav()` | 2063-2068 | Filter by role; add logout button |
| `navigate()` | 2076-2081 | Add audit call |
| `render()` | 2086-2099 | Add dept head and admin routing |
| `DATAPATH_SYSTEM_PROMPT` | 3851 | Parameterize for dept head scoping |

New sections inserted after `render()` (~line 2099):
- `USERS` config block (sentinel-delimited)
- `COMMENTARY` config block (sentinel-delimited)
- Auth utilities (`generateSalt`, `sha256WithSalt`, `escHtml`, `auditLog`)
- Dept head shell render functions
- Admin panel render functions
- Data scoping utilities
- Admin save flow

---

## Task 1: USERS & COMMENTARY config blocks + crypto utilities

**Files:**
- Modify: `monthly financial analysis.html` — insert after line 331 (end of existing auth script, before `<div class="app">`)

- [ ] **Step 1: Insert the USERS + COMMENTARY + utility script block**

Find the exact `</script>` at line 331 (end of the auth script). Insert the following new `<script>` block immediately after it (before `<div class="app" id="main-app">`):

```html
<script>
// @@USERS_START@@
const USERS = [
  {id:'scott',   name:'Scott Gordon',         username:'scott',   passwordHash:'', salt:'', role:'admin',     departments:[], views:[]},
  {id:'david',   name:'David Darmstandler',   username:'david',   passwordHash:'', salt:'', role:'admin',     departments:[], views:[]},
  {id:'james',   name:'James Bates',          username:'james',   passwordHash:'', salt:'', role:'admin',     departments:[], views:[]},
  {id:'swalski', name:'Stephen Walski',       username:'swalski', passwordHash:'', salt:'', role:'dept_head', departments:['professional_services','security'],         views:['overview','pl','gross_profit','labor']},
  {id:'brandon', name:'Brandon H.',           username:'brandon', passwordHash:'', salt:'', role:'dept_head', departments:['managed_services'],                         views:['overview','pl','gross_profit','labor']},
  {id:'timc',    name:'Tim C.',               username:'timc',    passwordHash:'', salt:'', role:'dept_head', departments:['r_and_d','ai_consulting'],                  views:['overview','pl','gross_profit','labor']},
  {id:'nathan',  name:'Nathan R.',            username:'nathan',  passwordHash:'', salt:'', role:'dept_head', departments:['managed_services','professional_services'],  views:['overview','pl','gross_profit','labor','sales']},
  {id:'dan',     name:'Dan S.',               username:'dan',     passwordHash:'', salt:'', role:'dept_head', departments:['managed_services'],                         views:['overview','pl','gross_profit','labor','sales','marketing']},
];
// @@USERS_END@@

// @@COMMENTARY_START@@
const COMMENTARY = {
  // key format: 'Mon YY:dept_key'  e.g. 'Mar 26:professional_services'
};
// @@COMMENTARY_END@@

// ─── HTML escape (always use for user-supplied strings in innerHTML) ──
function escHtml(str){
  return String(str)
    .replace(/&/g,'&amp;')
    .replace(/</g,'&lt;')
    .replace(/>/g,'&gt;')
    .replace(/"/g,'&quot;')
    .replace(/'/g,'&#39;');
}

// ─── Crypto utilities ────────────────────────────────────────────
function generateSalt(len=16){
  const arr=new Uint8Array(len);
  crypto.getRandomValues(arr);
  return Array.from(arr).map(b=>b.toString(16).padStart(2,'0')).join('');
}

async function sha256WithSalt(salt, pw){
  const enc = new TextEncoder().encode(salt + pw);
  const buf = await crypto.subtle.digest('SHA-256', enc);
  return Array.from(new Uint8Array(buf)).map(b=>b.toString(16).padStart(2,'0')).join('');
}

// ─── Audit log ───────────────────────────────────────────────────
const _auditMem = [];
function auditLog(userId, action, detail=''){
  const entry = {ts: new Date().toISOString(), userId, action, detail};
  _auditMem.push(entry);
  try {
    const stored = JSON.parse(localStorage.getItem('dp_audit_log')||'[]');
    stored.push(entry);
    if(stored.length > 500) stored.splice(0, stored.length-500);
    localStorage.setItem('dp_audit_log', JSON.stringify(stored));
  } catch(e){}
}
</script>
```

- [ ] **Step 2: Verify by opening the file in a browser**

Open `monthly financial analysis.html` in Chrome. Open DevTools console and run:
```js
USERS.length          // expected: 8
COMMENTARY            // expected: {}
generateSalt()        // expected: 32-char hex string
escHtml('<script>')   // expected: '&lt;script&gt;'
```

- [ ] **Step 3: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: add USERS/COMMENTARY config blocks, escHtml, and crypto utilities"
```

---

## Task 2: Replace auth gate with multi-user username + password login

**Files:**
- Modify: `monthly financial analysis.html` lines 277-331

The existing gate has one password field and a single `PW_HASH`. Replace the entire section.

- [ ] **Step 1: Replace the login gate HTML (lines 277-286)**

Find and replace:
```html
<div id="login-gate" style="display:flex;align-items:center;justify-content:center;height:100vh;background:#f8f9fa;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif">
  <div style="background:#fff;border:1px solid #e0e0e0;border-radius:12px;padding:40px;width:380px;box-shadow:0 4px 24px rgba(0,0,0,0.08);text-align:center">
    <div style="font-weight:800;font-size:20px;color:#1e40af;letter-spacing:-0.3px;margin-bottom:4px">DATAPATH</div>
    <div style="font-size:11px;color:#2563eb;text-transform:uppercase;letter-spacing:1.5px;margin-bottom:24px">Financial Command Center</div>
    <div style="font-size:14px;color:#6b7280;margin-bottom:16px">Enter password to continue</div>
    <input id="pw-input" type="password" placeholder="Password" style="width:100%;padding:12px 16px;border:1px solid #e0e0e0;border-radius:8px;font-size:14px;outline:none;margin-bottom:12px" onkeydown="if(event.key==='Enter')checkPw()">
    <button onclick="checkPw()" style="width:100%;padding:12px;background:#1e40af;color:#fff;border:none;border-radius:8px;font-size:14px;font-weight:600;cursor:pointer">Sign In</button>
    <div id="pw-error" style="color:#dc2626;font-size:13px;margin-top:10px;display:none">Incorrect password</div>
  </div>
</div>
```

With:
```html
<div id="login-gate" style="display:flex;align-items:center;justify-content:center;height:100vh;background:#f8f9fa;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif">
  <div style="background:#fff;border:1px solid #e0e0e0;border-radius:12px;padding:40px;width:380px;box-shadow:0 4px 24px rgba(0,0,0,0.08);text-align:center">
    <div style="font-weight:800;font-size:20px;color:#1e40af;letter-spacing:-0.3px;margin-bottom:4px">DATAPATH</div>
    <div style="font-size:11px;color:#2563eb;text-transform:uppercase;letter-spacing:1.5px;margin-bottom:24px">Financial Command Center</div>
    <input id="login-username" type="text" placeholder="Username" autocomplete="username" style="width:100%;padding:12px 16px;border:1px solid #e0e0e0;border-radius:8px;font-size:14px;outline:none;margin-bottom:10px;box-sizing:border-box" onkeydown="if(event.key==='Enter')document.getElementById('login-password').focus()">
    <input id="login-password" type="password" placeholder="Password" autocomplete="current-password" style="width:100%;padding:12px 16px;border:1px solid #e0e0e0;border-radius:8px;font-size:14px;outline:none;margin-bottom:14px;box-sizing:border-box" onkeydown="if(event.key==='Enter')checkLogin()">
    <button onclick="checkLogin()" style="width:100%;padding:12px;background:#1e40af;color:#fff;border:none;border-radius:8px;font-size:14px;font-weight:600;cursor:pointer">Sign In</button>
    <div id="login-error" style="color:#dc2626;font-size:13px;margin-top:10px;min-height:18px"></div>
  </div>
</div>
```

- [ ] **Step 2: Replace the entire auth script block (lines 287-331)**

Find and replace everything from `<script>` through the closing `</script>` of the auth block (the block that starts with `// Password stored as SHA-256 hash`):

```html
<script>
const AUTH_TIMEOUT = 60 * 60 * 1000; // 1 hour
const LOCKOUT_MAX  = 5;
const LOCKOUT_MS   = 15 * 60 * 1000; // 15 minutes

// ─── Session helpers ─────────────────────────────────────────────
function _sessionStore(role){ return role === 'admin' ? localStorage : sessionStorage; }

function saveSession(user){
  const tok = {userId:user.id, role:user.role, loginTime:Date.now(), lastActive:Date.now()};
  _sessionStore(user.role).setItem('dp_session', JSON.stringify(tok));
}
function loadSession(){
  let tok = null;
  try { tok = JSON.parse(localStorage.getItem('dp_session')); } catch(e){}
  if(!tok) try { tok = JSON.parse(sessionStorage.getItem('dp_session')); } catch(e){}
  return tok;
}
function clearSession(){
  localStorage.removeItem('dp_session');
  sessionStorage.removeItem('dp_session');
}
function refreshSession(){
  const tok = loadSession();
  if(!tok) return;
  tok.lastActive = Date.now();
  _sessionStore(tok.role).setItem('dp_session', JSON.stringify(tok));
}
function isSessionValid(){
  const tok = loadSession();
  if(!tok) return false;
  return (Date.now() - tok.lastActive) < AUTH_TIMEOUT;
}

// ─── Lockout helpers ─────────────────────────────────────────────
function _lockoutKey(username){ return 'dp_lockout_' + username; }
function getLockout(username){
  try { return JSON.parse(sessionStorage.getItem(_lockoutKey(username)) || 'null'); } catch(e){ return null; }
}
function setLockout(username, data){
  sessionStorage.setItem(_lockoutKey(username), JSON.stringify(data));
}
function checkLockout(username){
  const lk = getLockout(username);
  if(!lk) return null;
  if(lk.attempts >= LOCKOUT_MAX && (Date.now() - lk.ts) < LOCKOUT_MS){
    return Math.ceil((LOCKOUT_MS - (Date.now() - lk.ts)) / 60000); // minutes remaining
  }
  if((Date.now() - lk.ts) >= LOCKOUT_MS){ sessionStorage.removeItem(_lockoutKey(username)); return null; }
  return null;
}
function recordFailedAttempt(username){
  const lk = getLockout(username) || {attempts:0, ts:Date.now()};
  lk.attempts++;
  lk.ts = Date.now();
  setLockout(username, lk);
}
function clearLockout(username){ sessionStorage.removeItem(_lockoutKey(username)); }

// ─── App show/hide ───────────────────────────────────────────────
function showApp(){
  const user = state.currentUser;
  document.getElementById('login-gate').style.display='none';
  if(user && user.role === 'dept_head'){
    document.getElementById('dh-app').style.display='flex';
    document.getElementById('main-app').style.display='none';
  } else {
    document.getElementById('main-app').style.display='flex';
    document.getElementById('dh-app').style.display='none';
  }
}
function lockApp(){
  clearSession();
  if(typeof state !== 'undefined') state.currentUser = null;
  document.getElementById('main-app').style.display='none';
  document.getElementById('dh-app').style.display='none';
  document.getElementById('login-gate').style.display='flex';
  document.getElementById('login-username').value='';
  document.getElementById('login-password').value='';
  document.getElementById('login-error').textContent='';
}

// ─── Login ───────────────────────────────────────────────────────
async function checkLogin(){
  const username = (document.getElementById('login-username').value||'').trim().toLowerCase();
  const password = document.getElementById('login-password').value||'';
  const errEl    = document.getElementById('login-error');
  errEl.textContent = '';

  const remaining = checkLockout(username);
  if(remaining !== null){
    errEl.textContent = 'Too many attempts. Try again in ' + remaining + ' min.';
    auditLog(username, 'login_blocked', 'locked out');
    return;
  }

  const user = USERS.find(u => u.username === username);
  if(!user){
    errEl.textContent = 'Invalid username or password.';
    recordFailedAttempt(username);
    auditLog(username, 'login_fail', 'username not found');
    return;
  }

  if(!user.passwordHash || !user.salt){
    errEl.textContent = 'Account not configured. Contact an admin.';
    return;
  }

  const hash = await sha256WithSalt(user.salt, password);
  if(hash !== user.passwordHash){
    recordFailedAttempt(username);
    const lk   = getLockout(username);
    const left = LOCKOUT_MAX - (lk ? lk.attempts : 1);
    errEl.textContent = left > 0
      ? 'Incorrect password. ' + left + ' attempt' + (left===1?'':'s') + ' remaining.'
      : 'Account locked for 15 minutes.';
    auditLog(username, 'login_fail', 'wrong password');
    return;
  }

  clearLockout(username);
  saveSession(user);
  state.currentUser = user;
  auditLog(username, 'login_success', user.role);
  showApp();
  render();
}

// ─── Inactivity & auto-restore ───────────────────────────────────
['click','keydown','mousemove','scroll','touchstart'].forEach(evt=>{
  document.addEventListener(evt, ()=>{ if(isSessionValid()) refreshSession(); });
});
setInterval(()=>{
  const tok = loadSession();
  if(tok && !isSessionValid()){ auditLog(tok.userId,'session_expired','inactivity'); lockApp(); }
}, 60000);

document.addEventListener('DOMContentLoaded', ()=>{
  const tok = loadSession();
  if(tok && isSessionValid()){
    const user = USERS.find(u => u.id === tok.userId);
    if(user){ state.currentUser = user; showApp(); render(); }
    else lockApp();
  }
});
</script>
```

- [ ] **Step 3: Add `currentUser`, `dhDept`, `dhView`, `adminSubTab`, `adminSelUser` to the state object**

Find:
```js
const state = {
  view: 'dashboard',
  selDept: null,
  selMonth: -1, // -1 means "latest"; set after DATA is built
  sidebarCollapsed: false,
```

Replace with:
```js
const state = {
  view: 'dashboard',
  selDept: null,
  selMonth: -1, // -1 means "latest"; set after DATA is built
  sidebarCollapsed: false,
  currentUser: null,      // populated on login
  dhDept: 'all',          // dept head active dept filter: 'all' | dept_key
  dhView: 'overview',     // dept head active view tab
  dhChatMessages: null,   // dept head advisor chat history
  dhSystemPrompt: '',     // dept head scoped system prompt
  adminSubTab: 'users',   // admin panel sub-tab: 'users' | 'commentary' | 'audit'
  adminSelUser: null,     // id of user selected in admin panel
  adminComMonth: null,    // commentary editor selected month
  adminComDept: null,     // commentary editor selected dept
  adminAuditFilter: '',   // audit log user filter
```

- [ ] **Step 4: Add the `#dh-app` container to HTML**

Find:
```html
<div class="app" id="main-app" style="display:none">
```

Insert immediately before it:
```html
<div id="dh-app" style="display:none;flex-direction:column;height:100vh;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;background:#f8f9fa">
  <div id="dh-header"></div>
  <div id="dh-pills" style="background:#fff;border-bottom:1px solid #e0e0e0;padding:8px 20px;display:flex;gap:8px;align-items:center;overflow-x:auto"></div>
  <div id="dh-main" style="flex:1;overflow:auto;padding:24px 28px"></div>
</div>
```

- [ ] **Step 5: Verify in browser**

Open the file. Login screen shows two fields. Open console: `USERS[0].username` returns `'scott'`. Attempt bad password → error message. No crash.

- [ ] **Step 6: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: replace single-password gate with multi-user salted auth and lockout"
```

---

## Task 3: Admin tab in nav + logout button + render routing

**Files:**
- Modify: `monthly financial analysis.html` — `NAV` array, `renderNav()`, `render()`

- [ ] **Step 1: Add `adminOnly` flag to NAV and add Admin entry**

Find:
```js
const NAV = [
  {id:'dashboard', label:'Dashboard', icon:'📊'},
  {id:'departments', label:'Departments', icon:'🏢'},
  {id:'grossprofit', label:'Gross Profit', icon:'💰'},
  {id:'pnl', label:'P&L Detail', icon:'📄'},
  {id:'labor', label:'Labor Allocation', icon:'👥'},
  {id:'saleswon', label:'Sales Won', icon:'🏆'},
  {id:'advisor', label:'Intelligence', icon:'🧠'},
  {id:'data', label:'Import Data', icon:'📤'},
];
```

Replace with:
```js
const NAV = [
  {id:'dashboard',   label:'Dashboard',        icon:'📊', adminOnly:false},
  {id:'departments', label:'Departments',       icon:'🏢', adminOnly:false},
  {id:'grossprofit', label:'Gross Profit',      icon:'💰', adminOnly:false},
  {id:'pnl',         label:'P&L Detail',        icon:'📄', adminOnly:false},
  {id:'labor',       label:'Labor Allocation',  icon:'👥', adminOnly:false},
  {id:'saleswon',    label:'Sales Won',         icon:'🏆', adminOnly:false},
  {id:'advisor',     label:'Intelligence',      icon:'🧠', adminOnly:false},
  {id:'data',        label:'Import Data',       icon:'📤', adminOnly:false},
  {id:'admin',       label:'Admin',             icon:'⚙️',  adminOnly:true},
];
```

- [ ] **Step 2: Update `renderNav()` to filter by role and add logout**

Find:
```js
function renderNav() {
  const nav = document.getElementById('nav');
  nav.innerHTML = NAV.map(n => `<button class="nav-btn ${state.view===n.id||(n.id==='departments'&&state.selDept)?'active':''}" onclick="navigate('${n.id}')">
    <span>${n.icon}</span><span class="nav-label">${n.label}</span></button>`).join('');
  const footer = document.getElementById('sidebar-footer');
  footer.innerHTML = `Data: ${latest().month}<br>Annualized: ${fmt(latest().totRev*12)}`;
}
```

Replace with:
```js
function renderNav() {
  const nav     = document.getElementById('nav');
  const isAdmin = state.currentUser && state.currentUser.role === 'admin';
  const visible = NAV.filter(n => !n.adminOnly || isAdmin);
  nav.innerHTML = visible.map(n => `<button class="nav-btn ${state.view===n.id||(n.id==='departments'&&state.selDept)?'active':''}" onclick="navigate('${n.id}')">
    <span>${n.icon}</span><span class="nav-label">${n.label}</span></button>`).join('');
  const footer   = document.getElementById('sidebar-footer');
  const userName = state.currentUser ? escHtml(state.currentUser.name.split(' ')[0]) : '';
  footer.innerHTML = (isAdmin ? 'Data: ' + latest().month + '<br>Annualized: ' + fmt(latest().totRev*12) + '<br>' : '')
    + '<button onclick="doLogout()" style="margin-top:8px;width:100%;padding:5px;background:none;border:1px solid #e0e0e0;border-radius:6px;color:#6b7280;font-size:11px;cursor:pointer">Sign out (' + userName + ')</button>';
}

function doLogout(){
  const tok = loadSession();
  if(tok) auditLog(tok.userId, 'logout', '');
  lockApp();
}
```

- [ ] **Step 3: Update `render()` to branch by role**

Find:
```js
function render() {
  destroyCharts();
  renderNav();
  const main = document.getElementById('main');
  main.scrollTop = 0;
  if (state.view === 'dashboard') renderDashboard(main);
  else if (state.view === 'departments' && state.selDept) renderDrilldown(main);
  else if (state.view === 'departments') renderDeptList(main);
  else if (state.view === 'grossprofit') renderGrossProfit(main);
  else if (state.view === 'pnl') renderPnL(main);
  else if (state.view === 'labor') renderLaborAllocation(main);
  else if (state.view === 'saleswon') renderSalesWon(main);
  else if (state.view === 'advisor') renderAdvisor(main);
}
```

Replace with:
```js
function render() {
  const user = state.currentUser;
  if(!user){ lockApp(); return; }
  if(user.role === 'dept_head'){ renderDeptHead(); return; }

  destroyCharts();
  renderNav();
  const main = document.getElementById('main');
  main.scrollTop = 0;
  if      (state.view === 'dashboard')                    renderDashboard(main);
  else if (state.view === 'departments' && state.selDept) renderDrilldown(main);
  else if (state.view === 'departments')                  renderDeptList(main);
  else if (state.view === 'grossprofit')                  renderGrossProfit(main);
  else if (state.view === 'pnl')                          renderPnL(main);
  else if (state.view === 'labor')                        renderLaborAllocation(main);
  else if (state.view === 'saleswon')                     renderSalesWon(main);
  else if (state.view === 'advisor')                      renderAdvisor(main);
  else if (state.view === 'admin')                        renderAdmin(main);
}
```

- [ ] **Step 4: Add stub `renderAdmin` and `renderDeptHead`**

Insert immediately after the `render()` function:
```js
function renderAdmin(main){
  main.innerHTML = '<div style="padding:40px;color:#64748b">Admin panel loading...</div>';
}
function renderDeptHead(){
  document.getElementById('dh-main').innerHTML = '<div style="padding:40px;color:#64748b">Dept head view loading...</div>';
}
```

- [ ] **Step 5: Add audit call to `navigate()`**

Find:
```js
function navigate(view) {
  if (view === 'data') { showUploadModal(); return; }
  state.view = view;
  if (view !== 'departments') state.selDept = null;
  render();
}
```

Replace with:
```js
function navigate(view) {
  if (view === 'data') { showUploadModal(); return; }
  auditLog(state.currentUser?.id||'unknown', 'navigate', view);
  state.view = view;
  if (view !== 'departments') state.selDept = null;
  render();
}
```

- [ ] **Step 6: Verify**

Log in as admin (temporarily hardcode a hash via console — see Task 12 Step 4 for how). Sidebar shows Admin tab. Footer shows Sign out button. Click Admin → stub message appears. No errors in console.

- [ ] **Step 7: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: admin tab in nav, logout button, render routing by role"
```

---

## Task 4: Data scoping utilities

**Files:**
- Modify: `monthly financial analysis.html` — insert new utility section after `render()`

- [ ] **Step 1: Insert scoping utilities** (replace the `renderAdmin` / `renderDeptHead` stubs with real implementations and add these before them)

Insert after the `render()` block, before the stub functions:

```js
// ═══════════════════════════════════════════════════════════════
// DATA SCOPING — dept head views
// ═══════════════════════════════════════════════════════════════

function scopedRev(m, depts){
  return depts.reduce((s,d) => s + (m.rev && m.rev[d] ? m.rev[d] : 0), 0);
}

function scopedVendorCOGS(m, depts){
  return depts.reduce((s,d) => s + (m.cogs && m.cogs[d] ? m.cogs[d] : 0), 0);
}

function scopedLabor(m, depts){
  if(m.labor && m.labor.byDept){
    return depts.reduce((s,d) => s + (m.labor.byDept[d] || 0), 0);
  }
  // Legacy pre-2026: distribute direct+project labor by dept revenue share
  const totalRev = DEPTS.reduce((s,dep) => s + (m.rev && m.rev[dep.id] ? m.rev[dep.id] : 0), 0);
  if(!totalRev) return 0;
  const share    = totalRev ? scopedRev(m, depts) / totalRev : 0;
  const totalLab = m.labor ? (m.labor.direct||0) + (m.labor.project||0) : 0;
  return Math.round(totalLab * share);
}

function scopedCOGS(m, depts){ return scopedVendorCOGS(m, depts) + scopedLabor(m, depts); }
function scopedGP(m, depts){ return scopedRev(m, depts) - scopedCOGS(m, depts); }
function scopedGPpct(m, depts){ const r = scopedRev(m, depts); return r ? (scopedGP(m, depts) / r * 100) : 0; }

// Returns last `count` months of a metric as array of numbers
function sparklineData(depts, metric, count=6){
  return DATA.slice(-count).map(m => {
    if(metric === 'rev')   return scopedRev(m, depts);
    if(metric === 'gp')    return scopedGP(m, depts);
    if(metric === 'labor') return scopedLabor(m, depts);
    return 0;
  });
}

// Renders a tiny inline SVG sparkline
function renderSparkline(data, color='#3b82f6', w=80, h=24){
  if(!data || data.length < 2) return '';
  const min = Math.min(...data), max = Math.max(...data), range = max - min || 1;
  const pts  = data.map((v,i) => {
    const x = (i / (data.length-1)) * w;
    const y = h - ((v - min) / range) * (h - 4) - 2;
    return x.toFixed(1) + ',' + y.toFixed(1);
  }).join(' ');
  return '<svg width="' + w + '" height="' + h + '" style="display:block;overflow:visible"><polyline points="' + pts + '" fill="none" stroke="' + color + '" stroke-width="1.5" stroke-linejoin="round" stroke-linecap="round"/></svg>';
}

function userDepts(user){ return DEPTS.filter(d => user.departments.includes(d.id)); }
function activeDepts(user){ return state.dhDept === 'all' ? user.departments : [state.dhDept].filter(d => user.departments.includes(d)); }

function shortDeptName(name){
  return name.replace('Projects / Prof. Services','Prof. Services').replace('Security (MDR/SOC)','Security');
}
```

- [ ] **Step 2: Verify in console**

```js
const m = DATA[DATA.length-1];
scopedRev(m, ['managed_services'])          // matches m.rev.managed_services
scopedLabor(m, ['professional_services'])   // byDept value for PS
sparklineData(['security'], 'rev', 6)       // array of 6 numbers
renderSparkline([100,120,110,130,125,140])  // SVG string starting with '<svg'
escHtml('<script>alert(1)</script>')        // '&lt;script&gt;alert(1)&lt;/script&gt;'
```

- [ ] **Step 3: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: add dept scoping utilities (scopedRev/GP/Labor, sparklines)"
```

---

## Task 5: Dept head shell — header, dept pills, month selector, tabs

**Files:**
- Modify: `monthly financial analysis.html` — replace `renderDeptHead` stub

- [ ] **Step 1: Replace `renderDeptHead` stub with full implementation**

```js
const DH_VIEWS = [
  {id:'overview',     label:'Overview'},
  {id:'pl',           label:'P&L'},
  {id:'gross_profit', label:'Gross Profit'},
  {id:'labor',        label:'Labor'},
  {id:'sales',        label:'Sales'},
  {id:'marketing',    label:'Marketing'},
  {id:'intelligence', label:'Intelligence'},
];

function renderDeptHead(){
  const user  = state.currentUser;
  if(!user || user.role !== 'dept_head') return;
  const depts = userDepts(user);
  const midx  = state.selMonth === -1 ? DATA.length-1 : state.selMonth;
  const month = DATA[midx];
  const isPre = !!month.preliminary;

  // ── Header ───────────────────────────────────────────────────
  const roleLabel = depts.map(d=>shortDeptName(d.name)).join(' + ');
  document.getElementById('dh-header').innerHTML =
    '<div style="background:#1e40af;padding:12px 20px;display:flex;align-items:center;justify-content:space-between">'
    + '<div style="display:flex;align-items:center;gap:12px">'
    + '<span style="color:#fff;font-weight:800;font-size:16px;letter-spacing:-0.3px">DATAPATH</span>'
    + '<span style="background:rgba(255,255,255,0.18);color:#fff;font-size:11px;font-weight:600;padding:3px 10px;border-radius:10px">' + escHtml(roleLabel) + '</span>'
    + '</div>'
    + '<div style="display:flex;align-items:center;gap:10px">'
    + '<span style="color:rgba(255,255,255,0.8);font-size:12px">' + escHtml(user.name) + '</span>'
    + '<button onclick="window.print()" class="dh-no-print" style="background:rgba(255,255,255,0.12);border:1px solid rgba(255,255,255,0.25);color:#fff;padding:5px 12px;border-radius:6px;font-size:12px;cursor:pointer">&#x2B07; Report</button>'
    + '<button onclick="doLogout()" style="background:rgba(255,255,255,0.12);border:1px solid rgba(255,255,255,0.25);color:#fff;padding:5px 12px;border-radius:6px;font-size:12px;cursor:pointer">Sign out</button>'
    + '</div></div>';

  // ── Dept pills + month selector ───────────────────────────────
  const pills = depts.length > 1
    ? '<button onclick="dhSetDept(\'all\')" style="' + (state.dhDept==='all'?'background:#1e40af;color:#fff':'background:#f1f5f9;color:#374151') + ';border:none;padding:5px 14px;border-radius:12px;font-size:12px;font-weight:600;cursor:pointer">All Depts</button>'
      + depts.map(d=>'<button onclick="dhSetDept(\'' + d.id + '\')" style="' + (state.dhDept===d.id?'background:#1e40af;color:#fff':'background:#f1f5f9;color:#374151') + ';border:none;padding:5px 14px;border-radius:12px;font-size:12px;font-weight:600;cursor:pointer">' + escHtml(shortDeptName(d.name)) + '</button>').join('')
    : '';
  const monthOpts = DATA.map((m,i) =>
    '<option value="' + i + '"' + (i===midx?' selected':'') + '>' + escHtml(m.month) + (m.preliminary?' *':'') + '</option>'
  ).reverse().join('');
  const dataThrough = 'Data through ' + escHtml(month.month) + (isPre ? ' &middot; PRELIMINARY' : ' &middot; Final');

  document.getElementById('dh-pills').innerHTML = pills
    + '<div style="margin-left:auto;display:flex;align-items:center;gap:10px">'
    + '<span style="font-size:11px;color:#94a3b8">' + dataThrough + '</span>'
    + '<select onchange="dhSetMonth(this.value)" style="border:1px solid #e0e0e0;border-radius:6px;padding:4px 8px;font-size:12px;color:#374151;background:#fff">' + monthOpts + '</select>'
    + '</div>';

  // ── View tabs ─────────────────────────────────────────────────
  const allowedViews = DH_VIEWS.filter(v => user.views.includes(v.id) || v.id === 'intelligence');
  if(!allowedViews.find(v=>v.id===state.dhView)) state.dhView = allowedViews[0].id;
  const tabs = allowedViews.map(v =>
    '<button onclick="dhNavigate(\'' + v.id + '\')" style="' + (state.dhView===v.id?'color:#1e40af;border-bottom:2px solid #1e40af':'color:#64748b;border-bottom:2px solid transparent') + ';background:none;border-top:none;border-left:none;border-right:none;padding:10px 16px;font-size:13px;font-weight:600;cursor:pointer">' + escHtml(v.label) + '</button>'
  ).join('');

  // ── Render main area ─────────────────────────────────────────
  const mainEl = document.getElementById('dh-main');
  mainEl.innerHTML = '<div id="dh-view-tabs" style="border-bottom:1px solid #e0e0e0;margin-bottom:20px;margin-top:-4px;display:flex">' + tabs + '</div><div id="dh-view"></div>';
  auditLog(user.id, 'view', state.dhView);
  renderDHView(document.getElementById('dh-view'), user, month);
}

function dhSetDept(key){ state.dhDept = key; renderDeptHead(); }
function dhSetMonth(idx){ state.selMonth = parseInt(idx,10); renderDeptHead(); }
function dhNavigate(viewId){ state.dhView = viewId; renderDeptHead(); }
```

- [ ] **Step 2: Add stub `renderDHView`**

```js
function renderDHView(el, user, month){
  el.innerHTML = '<div style="color:#64748b;padding:20px">Loading...</div>';
}
```

- [ ] **Step 3: Test dept head shell**

Set a test hash for `swalski` via console:
```js
const salt = generateSalt();
sha256WithSalt(salt, 'test123').then(h => {
  const u = USERS.find(u=>u.id==='swalski');
  u.salt = salt; u.passwordHash = h;
  console.log('Done — log in as swalski / test123');
});
```

Log in as `swalski`. Should see: blue header with "Prof. Services + Security" badge, dept pills, month selector, tab row (Overview, P&L, Gross Profit, Labor, Intelligence), "Loading..." in view area.

- [ ] **Step 4: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: dept head shell — header, dept pills, month selector, tab navigation"
```

---

## Task 6: Dept head KPI cards, sparklines, breakdown table

**Files:**
- Modify: `monthly financial analysis.html` — replace `renderDHView` stub

- [ ] **Step 1: Replace `renderDHView` stub**

```js
function renderDHView(el, user, month){
  if(state.dhView && state.dhView !== 'overview'){
    renderDHViewByTab(el, user, month, state.dhView);
    return;
  }
  const depts = activeDepts(user);
  const isPre = !!month.preliminary;
  const midx  = state.selMonth === -1 ? DATA.length-1 : state.selMonth;
  const prev  = midx > 0 ? DATA[midx-1] : null;

  const rev  = scopedRev(month, depts);
  const gp   = scopedGP(month, depts);
  const gpPct= scopedGPpct(month, depts);
  const lab  = scopedLabor(month, depts);

  function momDelta(curr, fn){
    if(!prev) return '';
    const pc = fn(prev);
    if(!pc) return '';
    const d     = ((curr - pc)/pc*100).toFixed(1);
    const color = curr >= pc ? '#16a34a' : '#dc2626';
    return '<div style="font-size:11px;color:' + color + '">' + (curr>=pc?'&#x2191;':'&#x2193;') + ' ' + Math.abs(d) + '% MoM</div>';
  }

  function budgetRow(val, budgetKey){
    if(!month.budget) return '';
    const b = depts.reduce((s,d)=>{ const bd=month.budget[d]; return s+(bd&&bd[budgetKey]?bd[budgetKey]:0); },0);
    if(!b) return '';
    const diff = val - b;
    const color = diff >= 0 ? '#16a34a' : '#dc2626';
    return '<div style="font-size:11px;color:' + color + '">vs budget: ' + (diff>=0?'+':'') + fmt(diff) + '</div>';
  }

  // Commentary lookup
  const noteText = (() => {
    for(const d of depts){ const key=month.month+':'+d; if(COMMENTARY[key]) return COMMENTARY[key]; }
    return null;
  })();

  const preBadge = isPre
    ? '<div style="background:#fef3c7;border:1px solid #fcd34d;border-radius:8px;padding:10px 14px;margin-bottom:20px;font-size:12px;color:#92400e"><strong>PRELIMINARY</strong> — Numbers not yet finalized by Citrin Cooperman.</div>'
    : '';
  const commentaryHtml = noteText
    ? '<div style="background:#fef3c7;border:1px solid #fcd34d;border-radius:8px;padding:14px 16px;margin-bottom:20px">'
      + '<div style="font-size:11px;font-weight:700;color:#92400e;margin-bottom:4px">NOTE FROM LEADERSHIP</div>'
      + '<div style="font-size:13px;color:#78350f">' + escHtml(noteText) + '</div></div>'
    : '';

  function kpiCard(label, valHtml, border, extra, spark){
    return '<div style="background:#fff;border:1px solid #e0e0e0;border-left:4px solid ' + border + ';border-radius:8px;padding:16px 18px;box-shadow:0 1px 3px rgba(0,0,0,0.04)">'
      + '<div style="font-size:11px;color:#64748b;font-weight:600;text-transform:uppercase;letter-spacing:0.5px;margin-bottom:6px">' + label + '</div>'
      + '<div style="font-size:22px;font-weight:700;color:#111;margin-bottom:4px">' + valHtml + '</div>'
      + extra
      + '<div style="margin-top:8px">' + spark + '</div></div>';
  }

  const gpValHtml = fmt(gp) + ' <span style="font-size:13px;color:#64748b">(' + gpPct.toFixed(1) + '%)</span>';
  const cards = '<div class="dh-kpi-grid" style="display:grid;grid-template-columns:repeat(3,1fr);gap:16px;margin-bottom:24px">'
    + kpiCard('Revenue',     fmt(rev),    '#3b82f6', momDelta(rev,  m=>scopedRev(m,depts))   + budgetRow(rev,'revenue'),  renderSparkline(sparklineData(depts,'rev',6),'#3b82f6'))
    + kpiCard('Gross Profit',gpValHtml,   '#16a34a', momDelta(gp,   m=>scopedGP(m,depts))    + budgetRow(gp,'gp'),        renderSparkline(sparklineData(depts,'gp',6),'#16a34a'))
    + kpiCard('Labor',       fmt(lab),    '#f59e0b', momDelta(lab,  m=>scopedLabor(m,depts)),                              renderSparkline(sparklineData(depts,'labor',6),'#f59e0b'))
    + '</div>';

  // Per-dept breakdown table (multi-dept + 'all' selected)
  let tableHtml = '';
  if(user.departments.length > 1 && state.dhDept === 'all'){
    const rows = userDepts(user).map(d => {
      const dr  = scopedRev(month,[d.id]);
      const dgp = scopedGP(month,[d.id]);
      const dpct= dr ? (dgp/dr*100).toFixed(1) : null;
      const color= dpct !== null && parseFloat(dpct) >= (BM[d.id]?.tgt||50) ? '#16a34a' : '#f59e0b';
      return '<tr>'
        + '<td style="padding:8px 12px;font-weight:600;color:#374151">' + escHtml(shortDeptName(d.name)) + '</td>'
        + '<td style="padding:8px 12px;color:#374151">' + fmt(dr) + '</td>'
        + '<td style="padding:8px 12px;color:#374151">' + fmt(dgp) + '</td>'
        + '<td style="padding:8px 12px;color:' + color + ';font-weight:600">' + (dpct !== null ? dpct+'%' : '&#x2014;') + '</td>'
        + '</tr>';
    }).join('');
    tableHtml = '<div style="background:#fff;border:1px solid #e0e0e0;border-radius:8px;overflow:hidden;box-shadow:0 1px 3px rgba(0,0,0,0.04)">'
      + '<table style="width:100%;border-collapse:collapse">'
      + '<thead><tr style="background:#f8fafc;border-bottom:1px solid #e0e0e0">'
      + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">Department</th>'
      + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">Revenue</th>'
      + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">Gross Profit</th>'
      + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">GP%</th>'
      + '</tr></thead><tbody>' + rows + '</tbody></table></div>';
  }

  el.innerHTML = preBadge + commentaryHtml + cards + tableHtml;
}
```

- [ ] **Step 2: Verify as `swalski`**

- Three KPI cards with blue/green/amber borders
- Sparklines visible under each number
- Dept table showing PS and Security rows
- PRELIMINARY badge on Mar 26
- Add a COMMENTARY entry via console: `COMMENTARY['Mar 26:professional_services'] = 'Test note';` — amber callout should appear

- [ ] **Step 3: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: dept head KPI cards with sparklines, per-dept table, commentary callout"
```

---

## Task 7: Dept head view tabs (P&L, GP, Labor, Sales, Marketing, Intelligence)

**Files:**
- Modify: `monthly financial analysis.html` — add `renderDHViewByTab` and `renderDHAdvisor`

- [ ] **Step 1: Add `renderDHViewByTab` after `renderDHView`**

```js
function renderDHViewByTab(el, user, month, viewId){
  const depts = activeDepts(user);
  const isPre = !!month.preliminary;
  const preBadge = isPre ? '<div style="background:#fef3c7;border:1px solid #fcd34d;border-radius:8px;padding:10px 14px;margin-bottom:16px;font-size:12px;color:#92400e"><strong>PRELIMINARY</strong></div>' : '';

  if(viewId === 'pl'){
    const rows = userDepts(user).filter(d=>depts.includes(d.id)).map(d=>{
      const r   = scopedRev(month,[d.id]);
      const lab = scopedLabor(month,[d.id]);
      const vc  = scopedVendorCOGS(month,[d.id]);
      const gp  = r - vc - lab;
      return '<tr style="background:#f8fafc"><td colspan="2" style="padding:8px 12px;font-weight:700;color:#374151">' + escHtml(shortDeptName(d.name)) + '</td></tr>'
        + '<tr><td style="padding:6px 12px 6px 24px;color:#64748b">Revenue</td><td style="padding:6px 12px;text-align:right">' + fmt(r) + '</td></tr>'
        + '<tr><td style="padding:6px 12px 6px 24px;color:#64748b">Vendor COGS</td><td style="padding:6px 12px;text-align:right">' + fmt(vc) + '</td></tr>'
        + '<tr><td style="padding:6px 12px 6px 24px;color:#64748b">Labor</td><td style="padding:6px 12px;text-align:right">' + fmt(lab) + '</td></tr>'
        + '<tr style="border-top:1px solid #e0e0e0"><td style="padding:8px 12px 8px 24px;font-weight:600">Gross Profit</td><td style="padding:8px 12px;text-align:right;font-weight:700;color:' + (gp>=0?'#16a34a':'#dc2626') + '">' + fmt(gp) + '</td></tr>';
    }).join('');
    el.innerHTML = preBadge + '<div style="background:#fff;border:1px solid #e0e0e0;border-radius:8px;overflow:hidden"><table style="width:100%;border-collapse:collapse">' + rows + '</table></div>';

  } else if(viewId === 'gross_profit'){
    const bars = userDepts(user).filter(d=>depts.includes(d.id)).map(d=>{
      const r   = scopedRev(month,[d.id]);
      const gp  = scopedGP(month,[d.id]);
      const pct = r ? gp/r*100 : 0;
      const tgt = BM[d.id]?.tgt||50;
      const color = pct >= tgt ? '#16a34a' : pct >= tgt*0.85 ? '#f59e0b' : '#dc2626';
      return '<div style="background:#fff;border:1px solid #e0e0e0;border-radius:8px;padding:16px 20px;margin-bottom:12px">'
        + '<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:10px">'
        + '<span style="font-weight:600;color:#374151">' + escHtml(shortDeptName(d.name)) + '</span>'
        + '<span style="font-weight:700;font-size:18px;color:' + color + '">' + pct.toFixed(1) + '%</span></div>'
        + '<div style="background:#f1f5f9;border-radius:4px;height:8px"><div style="background:' + color + ';border-radius:4px;height:8px;width:' + Math.min(pct,100).toFixed(1) + '%"></div></div>'
        + '<div style="display:flex;justify-content:space-between;margin-top:6px;font-size:11px;color:#94a3b8">'
        + '<span>GP: ' + fmt(gp) + ' of ' + fmt(r) + '</span><span>Target: ' + tgt + '%</span></div></div>';
    }).join('');
    el.innerHTML = preBadge + bars;

  } else if(viewId === 'labor'){
    const rows = userDepts(user).filter(d=>depts.includes(d.id)).map(d=>{
      const lab = scopedLabor(month,[d.id]);
      const rev = scopedRev(month,[d.id]);
      const pct = rev ? (lab/rev*100).toFixed(1)+'%' : '&#x2014;';
      return '<tr>'
        + '<td style="padding:10px 12px;font-weight:600;color:#374151">' + escHtml(shortDeptName(d.name)) + '</td>'
        + '<td style="padding:10px 12px;color:#374151">' + fmt(lab) + '</td>'
        + '<td style="padding:10px 12px;color:#64748b">' + pct + ' of revenue</td></tr>';
    }).join('');
    el.innerHTML = preBadge + '<div style="background:#fff;border:1px solid #e0e0e0;border-radius:8px;overflow:hidden">'
      + '<table style="width:100%;border-collapse:collapse">'
      + '<thead><tr style="background:#f8fafc;border-bottom:1px solid #e0e0e0">'
      + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">Department</th>'
      + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">Labor Cost</th>'
      + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">% of Revenue</th>'
      + '</tr></thead><tbody>' + rows + '</tbody></table></div>';

  } else if(viewId === 'sales'){
    el.innerHTML = preBadge + '<div id="dh-sales-inner"></div>';
    setTimeout(()=>{ const tmp=document.createElement('div'); renderSalesWon(tmp); document.getElementById('dh-sales-inner').appendChild(tmp); },0);

  } else if(viewId === 'marketing'){
    const mktSpend = month.opex ? (month.opex.marketing||0) : 0;
    el.innerHTML = preBadge + '<div style="background:#fff;border:1px solid #e0e0e0;border-radius:8px;padding:24px">'
      + '<div style="font-size:11px;color:#64748b;font-weight:600;text-transform:uppercase;margin-bottom:6px">Marketing Spend &mdash; ' + escHtml(month.month) + '</div>'
      + '<div style="font-size:28px;font-weight:700;color:#374151">' + fmt(mktSpend) + '</div>'
      + '<div style="font-size:12px;color:#94a3b8;margin-top:4px">From OpEx marketing bucket</div></div>';

  } else if(viewId === 'intelligence'){
    renderDHAdvisor(el, user);

  } else {
    el.innerHTML = '<div style="color:#94a3b8;padding:20px">View not available.</div>';
  }
}
```

- [ ] **Step 2: Add `renderDHAdvisor` and `dhSendMessage`**

```js
function renderDHAdvisor(el, user){
  const depts     = userDepts(user);
  const deptNames = depts.map(d=>d.name).join(' and ');
  const last3     = DATA.slice(-3);
  const scopedCtx = last3.map(m=>{
    const d2 = user.departments;
    return m.month + ': Revenue ' + fmt(scopedRev(m,d2)) + ', GP ' + fmt(scopedGP(m,d2)) + ' (' + scopedGPpct(m,d2).toFixed(1) + '%), Labor ' + fmt(scopedLabor(m,d2));
  }).join('\n');

  state.dhSystemPrompt = 'You are a financial advisor for ' + user.name + ' at Datapath, focused exclusively on the ' + deptNames + ' department' + (depts.length>1?'s':'') + '.\n\n'
    + 'IMPORTANT RESTRICTIONS:\n'
    + '- Only discuss data for: ' + deptNames + '\n'
    + '- Do NOT reveal company-wide revenue, total EBITDA, overall company profit, or any other department\'s numbers\n'
    + '- If asked about company-wide metrics or other departments, politely explain you can only discuss their assigned departments\n'
    + '- Do NOT reveal headcount numbers or individual salaries\n\n'
    + 'THEIR RECENT DATA:\n' + scopedCtx + '\n\n'
    + 'MSP BENCHMARKS:\n'
    + depts.map(d=>'- ' + d.name + ': GP target ' + (BM[d.id]?.tgt||50) + '% (top ' + (BM[d.id]?.top||55) + '%, median ' + (BM[d.id]?.med||45) + '%)').join('\n') + '\n\n'
    + 'Be direct, use their actual numbers, compare to benchmarks. Frame advice in terms of what they can improve in their department.';

  if(!state.dhChatMessages){
    state.dhChatMessages = [{role:'assistant', content:'Hi ' + user.name.split(' ')[0] + '! I can help you understand your ' + depts.map(d=>shortDeptName(d.name)).join(' and ') + ' financial performance. What would you like to know?'}];
  }

  el.innerHTML = '<div style="display:flex;flex-direction:column;height:calc(100vh - 240px)">'
    + '<div id="dh-chat-msgs" style="flex:1;overflow-y:auto;padding:4px 0;display:flex;flex-direction:column;gap:12px"></div>'
    + '<div style="display:flex;gap:8px;padding-top:12px;border-top:1px solid #e0e0e0;margin-top:12px">'
    + '<input id="dh-chat-input" type="text" placeholder="Ask about your department financials..." style="flex:1;padding:10px 14px;border:1px solid #e0e0e0;border-radius:8px;font-size:13px;outline:none" onkeydown="if(event.key===\'Enter\')dhSendMessage()">'
    + '<button onclick="dhSendMessage()" style="background:#1e40af;color:#fff;border:none;border-radius:8px;padding:10px 18px;font-size:13px;font-weight:600;cursor:pointer">Send</button>'
    + '</div></div>';
  renderDHChatMessages();
}

function renderDHChatMessages(){
  const el = document.getElementById('dh-chat-msgs');
  if(!el) return;
  el.innerHTML = (state.dhChatMessages||[]).map(m=>{
    const isUser = m.role === 'user';
    return '<div style="display:flex;' + (isUser?'justify-content:flex-end':'') + '">'
      + '<div style="max-width:80%;background:' + (isUser?'#1e40af':'#f8fafc') + ';color:' + (isUser?'#fff':'#374151') + ';border:1px solid ' + (isUser?'#1e40af':'#e0e0e0') + ';border-radius:10px;padding:10px 14px;font-size:13px;line-height:1.5;white-space:pre-wrap">'
      + escHtml(m.content) + '</div></div>';
  }).join('');
  el.scrollTop = el.scrollHeight;
}

async function dhSendMessage(){
  const input = document.getElementById('dh-chat-input');
  const text  = (input.value||'').trim();
  if(!text) return;
  input.value = '';
  if(!state.dhChatMessages) state.dhChatMessages = [];
  state.dhChatMessages.push({role:'user', content:text});
  renderDHChatMessages();

  const apiKey = localStorage.getItem('dp_api_key');
  if(!apiKey){
    state.dhChatMessages.push({role:'assistant', content:'Please have an admin set up the API key in the Intelligence tab first.'});
    renderDHChatMessages();
    return;
  }
  try {
    const res = await fetch('https://api.anthropic.com/v1/messages',{
      method:'POST',
      headers:{'Content-Type':'application/json','x-api-key':apiKey,'anthropic-version':'2023-06-01','anthropic-dangerous-direct-browser-access':'true'},
      body: JSON.stringify({model:'claude-sonnet-4-20250514', max_tokens:1024, system:state.dhSystemPrompt, messages:state.dhChatMessages})
    });
    const data = await res.json();
    state.dhChatMessages.push({role:'assistant', content:data.content?.[0]?.text||'No response.'});
  } catch(e){
    state.dhChatMessages.push({role:'assistant', content:'Error contacting AI. Check your API key.'});
  }
  renderDHChatMessages();
}
```

- [ ] **Step 3: Verify each tab**

As `swalski` (PS + Security): click P&L → rows for both depts. GP tab → progress bars with target line. Labor → cost table. Intelligence → chat UI appears, greeting message shown.

As `nathan`: Sales tab appears and shows the sales won table.

As `dan`: Marketing tab shows marketing spend from `month.opex.marketing`.

- [ ] **Step 4: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: dept head P&L, GP, Labor, Sales, Marketing, and Intelligence tabs"
```

---

## Task 8: PDF export + mobile styles

**Files:**
- Modify: `monthly financial analysis.html` — `<style>` block

- [ ] **Step 1: Add print and mobile CSS to the existing `<style>` block**

Find the closing `</style>` tag of the main style block (near line 275) and insert before it:

```css
/* ── PDF export ────────────────────────────────────────── */
@media print {
  #dh-header, #dh-pills { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
  body > *:not(#dh-app) { display: none !important; }
  #dh-app { display: flex !important; }
  #dh-view-tabs, .dh-no-print { display: none !important; }
  #dh-header button { display: none !important; }
  #dh-main { padding: 0 !important; }
}

/* ── Dept head mobile ──────────────────────────────────── */
@media (max-width: 640px) {
  #dh-header > div { flex-wrap: wrap; gap: 8px; }
  .dh-kpi-grid { grid-template-columns: 1fr !important; }
  #dh-pills { flex-wrap: nowrap; overflow-x: auto; -webkit-overflow-scrolling: touch; }
  #dh-main { padding: 16px !important; }
}
```

- [ ] **Step 2: Verify print**

Log in as dept head. Click "⬇ Report". Chrome print dialog opens. In preview:
- Blue header visible with color
- KPI cards and table visible
- No sign-out or pill buttons in print output

Use DevTools device emulator at 375px wide. KPI cards stack to single column. Dept pills scroll horizontally.

- [ ] **Step 3: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: PDF export via window.print(), mobile-responsive dept head shell"
```

---

## Task 9: Admin panel — User Management (master/detail)

**Files:**
- Modify: `monthly financial analysis.html` — replace `renderAdmin` stub

- [ ] **Step 1: Replace `renderAdmin` stub**

```js
function renderAdmin(main){
  const subTab = state.adminSubTab || 'users';
  const tabBar = ['users','commentary','audit'].map(t =>
    '<button onclick="state.adminSubTab=\'' + t + '\';renderAdmin(document.getElementById(\'main\'))" style="' + (subTab===t?'color:#1e40af;border-bottom:2px solid #1e40af':'color:#64748b;border-bottom:2px solid transparent') + ';background:none;border-top:none;border-left:none;border-right:none;padding:10px 16px;font-size:13px;font-weight:600;cursor:pointer">'
    + {users:'Users',commentary:'Commentary',audit:'Audit Log'}[t] + '</button>'
  ).join('');

  main.innerHTML = '<div style="padding:24px 28px">'
    + '<div style="font-size:20px;font-weight:700;color:#111;margin-bottom:16px">Admin</div>'
    + '<div style="border-bottom:1px solid #e0e0e0;margin-bottom:24px;display:flex">' + tabBar + '</div>'
    + '<div id="admin-content"></div></div>';

  const content = document.getElementById('admin-content');
  if(subTab === 'users')           renderAdminUsers(content);
  else if(subTab === 'commentary') renderAdminCommentary(content);
  else if(subTab === 'audit')      renderAdminAudit(content);
}
```

- [ ] **Step 2: Add `renderAdminUsers` and `renderAdminUserPanel`**

```js
function renderAdminUsers(el){
  const deptHeads = USERS.filter(u => u.role === 'dept_head');
  if(!state.adminSelUser && deptHeads[0]) state.adminSelUser = deptHeads[0].id;
  const selId   = state.adminSelUser;
  const selUser = USERS.find(u => u.id === selId);

  const sidebar = deptHeads.map(u =>
    '<div onclick="state.adminSelUser=\'' + u.id + '\';renderAdmin(document.getElementById(\'main\'))" style="padding:10px 14px;border-left:3px solid ' + (u.id===selId?'#1e40af':'transparent') + ';background:' + (u.id===selId?'#eff6ff':'transparent') + ';cursor:pointer">'
    + '<div style="font-weight:' + (u.id===selId?'700':'600') + ';color:' + (u.id===selId?'#1e40af':'#374151') + ';font-size:13px">' + escHtml(u.name) + '</div>'
    + '<div style="font-size:11px;color:#94a3b8">' + escHtml(u.username) + ' &middot; ' + escHtml(userDepts(u).map(d=>shortDeptName(d.name)).join(', ')) + '</div></div>'
  ).join('');

  el.innerHTML = '<div style="display:grid;grid-template-columns:220px 1fr;gap:0;background:#fff;border:1px solid #e0e0e0;border-radius:8px;overflow:hidden;min-height:400px">'
    + '<div style="border-right:1px solid #e0e0e0">'
    + '<div style="padding:10px 14px;font-size:11px;font-weight:700;color:#64748b;letter-spacing:0.5px;text-transform:uppercase;border-bottom:1px solid #f1f5f9">Users (' + deptHeads.length + ')</div>'
    + sidebar
    + '<div style="padding:10px 14px;border-top:1px solid #f1f5f9;margin-top:8px"><button onclick="addNewUser()" style="color:#2563eb;background:none;border:none;font-size:12px;font-weight:600;cursor:pointer">+ Add User</button></div>'
    + '</div>'
    + '<div style="padding:20px">' + (selUser ? renderAdminUserPanel(selUser) : '<div style="color:#94a3b8;padding:20px">Select a user</div>') + '</div>'
    + '</div>';
}

function renderAdminUserPanel(user){
  const allViews = [{id:'overview',label:'Overview'},{id:'pl',label:'P&L'},{id:'gross_profit',label:'Gross Profit'},{id:'labor',label:'Labor'},{id:'sales',label:'Sales'},{id:'marketing',label:'Marketing'}];

  const deptPills = DEPTS.map(d =>
    '<button onclick="toggleUserDept(\'' + user.id + '\',\'' + d.id + '\')" style="' + (user.departments.includes(d.id)?'background:#dbeafe;color:#1e40af':'background:#f1f5f9;color:#94a3b8') + ';border:none;padding:4px 10px;border-radius:10px;font-size:12px;font-weight:600;cursor:pointer;margin:2px">'
    + (user.departments.includes(d.id)?'&#10003; ':'') + escHtml(shortDeptName(d.name)) + '</button>'
  ).join('');

  const viewPills = allViews.map(v =>
    '<button onclick="toggleUserView(\'' + user.id + '\',\'' + v.id + '\')" style="' + (user.views.includes(v.id)?'background:#dbeafe;color:#1e40af':'background:#f1f5f9;color:#94a3b8') + ';border:none;padding:4px 10px;border-radius:10px;font-size:12px;font-weight:600;cursor:pointer;margin:2px">'
    + (user.views.includes(v.id)?'&#10003; ':'') + escHtml(v.label) + '</button>'
  ).join('');

  return '<div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:16px">'
    + '<div style="font-size:16px;font-weight:700;color:#111">' + escHtml(user.name) + '</div>'
    + '<button onclick="removeUser(\'' + user.id + '\')" style="background:#fee2e2;color:#dc2626;border:none;padding:5px 12px;border-radius:6px;font-size:12px;font-weight:600;cursor:pointer">Remove</button>'
    + '</div>'
    + '<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:16px">'
    + '<div><div style="font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase;margin-bottom:4px">Username</div>'
    + '<input id="edit-username-' + user.id + '" value="' + escHtml(user.username) + '" style="width:100%;padding:8px 10px;border:1px solid #d1d5db;border-radius:6px;font-size:13px;box-sizing:border-box"></div>'
    + '<div><div style="font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase;margin-bottom:4px">New Password <span style="font-weight:400">(blank = keep)</span></div>'
    + '<input id="edit-password-' + user.id + '" type="password" placeholder="Enter new password..." style="width:100%;padding:8px 10px;border:1px solid #d1d5db;border-radius:6px;font-size:13px;box-sizing:border-box"></div>'
    + '</div>'
    + '<div style="margin-bottom:14px"><div style="font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase;margin-bottom:6px">Departments</div><div>' + deptPills + '</div></div>'
    + '<div style="margin-bottom:20px"><div style="font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase;margin-bottom:6px">Views</div><div>' + viewPills + '</div></div>'
    + '<div style="display:flex;justify-content:flex-end"><button onclick="saveUser(\'' + user.id + '\')" style="background:#1e40af;color:#fff;border:none;padding:8px 20px;border-radius:6px;font-size:13px;font-weight:600;cursor:pointer">Save Changes</button></div>';
}

function toggleUserDept(userId, deptId){
  const user = USERS.find(u=>u.id===userId); if(!user) return;
  const idx = user.departments.indexOf(deptId);
  if(idx>=0) user.departments.splice(idx,1); else user.departments.push(deptId);
  renderAdmin(document.getElementById('main'));
}
function toggleUserView(userId, viewId){
  const user = USERS.find(u=>u.id===userId); if(!user) return;
  const idx = user.views.indexOf(viewId);
  if(idx>=0) user.views.splice(idx,1); else user.views.push(viewId);
  renderAdmin(document.getElementById('main'));
}
async function saveUser(userId){
  const user = USERS.find(u=>u.id===userId); if(!user) return;
  const newUser = (document.getElementById('edit-username-'+userId)?.value||'').trim().toLowerCase();
  const newPass = document.getElementById('edit-password-'+userId)?.value||'';
  if(newUser) user.username = newUser;
  if(newPass){ user.salt = generateSalt(); user.passwordHash = await sha256WithSalt(user.salt, newPass); }
  auditLog(state.currentUser?.id||'admin', 'user_saved', userId);
  await saveToFile();
  renderAdmin(document.getElementById('main'));
}
function removeUser(userId){
  if(!confirm('Remove user ' + userId + '? This cannot be undone.')) return;
  const idx = USERS.findIndex(u=>u.id===userId);
  if(idx>=0) USERS.splice(idx,1);
  state.adminSelUser = USERS.find(u=>u.role==='dept_head')?.id||null;
  saveToFile().then(()=>renderAdmin(document.getElementById('main')));
}
function addNewUser(){
  const id = 'user_'+Date.now();
  USERS.push({id, name:'New User', username:'newuser', passwordHash:'', salt:'', role:'dept_head', departments:[], views:['overview']});
  state.adminSelUser = id;
  renderAdmin(document.getElementById('main'));
}
```

- [ ] **Step 2: Verify**

Log in as admin. Click Admin → Users. Left sidebar shows 5 dept heads. Click Stephen Walski → right panel shows username, password field, dept pills (PS and Security checked blue), view pills (Overview/P&L/GP/Labor checked). Toggle a dept — color changes immediately.

- [ ] **Step 3: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: admin user management panel with dept/view toggles and password reset"
```

---

## Task 10: Admin Commentary editor + Audit Log view

**Files:**
- Modify: `monthly financial analysis.html` — add `renderAdminCommentary`, `renderAdminAudit`

- [ ] **Step 1: Add Commentary editor**

Insert after `addNewUser()`:
```js
function renderAdminCommentary(el){
  const months   = DATA.map(m=>m.month).reverse();
  const selMonth = state.adminComMonth || months[0];
  const selDept  = state.adminComDept  || DEPTS[0].id;
  const key      = selMonth + ':' + selDept;
  const current  = COMMENTARY[key] || '';

  el.innerHTML = '<div style="background:#fff;border:1px solid #e0e0e0;border-radius:8px;padding:24px;max-width:700px">'
    + '<div style="font-size:15px;font-weight:700;color:#111;margin-bottom:16px">Add / Edit Notes for Dept Heads</div>'
    + '<div style="display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:16px">'
    + '<div><div style="font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase;margin-bottom:4px">Month</div>'
    + '<select onchange="state.adminComMonth=this.value;renderAdmin(document.getElementById(\'main\'))" style="width:100%;padding:8px 10px;border:1px solid #d1d5db;border-radius:6px;font-size:13px">'
    + months.map(m=>'<option value="' + escHtml(m) + '"' + (m===selMonth?' selected':'') + '>' + escHtml(m) + '</option>').join('')
    + '</select></div>'
    + '<div><div style="font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase;margin-bottom:4px">Department</div>'
    + '<select onchange="state.adminComDept=this.value;renderAdmin(document.getElementById(\'main\'))" style="width:100%;padding:8px 10px;border:1px solid #d1d5db;border-radius:6px;font-size:13px">'
    + DEPTS.map(d=>'<option value="' + d.id + '"' + (d.id===selDept?' selected':'') + '>' + escHtml(shortDeptName(d.name)) + '</option>').join('')
    + '</select></div></div>'
    + '<div style="font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase;margin-bottom:4px">Note (shown as amber callout to dept head)</div>'
    + '<textarea id="commentary-input" rows="4" style="width:100%;padding:10px;border:1px solid #d1d5db;border-radius:6px;font-size:13px;box-sizing:border-box;resize:vertical">' + escHtml(current) + '</textarea>'
    + '<div style="display:flex;justify-content:space-between;align-items:center;margin-top:12px">'
    + (current ? '<button onclick="clearCommentary(\'' + key + '\')" style="color:#dc2626;background:none;border:none;font-size:12px;cursor:pointer">Clear note</button>' : '<div></div>')
    + '<button onclick="saveCommentary(\'' + key + '\')" style="background:#1e40af;color:#fff;border:none;padding:8px 20px;border-radius:6px;font-size:13px;font-weight:600;cursor:pointer">Save Note</button>'
    + '</div></div>';
}

async function saveCommentary(key){
  const text = (document.getElementById('commentary-input')?.value||'').trim();
  if(text) COMMENTARY[key] = text; else delete COMMENTARY[key];
  auditLog(state.currentUser?.id||'admin', 'commentary_saved', key);
  await saveToFile();
  renderAdmin(document.getElementById('main'));
}
function clearCommentary(key){
  delete COMMENTARY[key];
  saveToFile().then(()=>renderAdmin(document.getElementById('main')));
}
```

- [ ] **Step 2: Add Audit Log view**

Insert after `clearCommentary`:
```js
function renderAdminAudit(el){
  let entries = [];
  try { entries = JSON.parse(localStorage.getItem('dp_audit_log')||'[]'); } catch(e){}
  entries = entries.slice().reverse();
  const filterUser = state.adminAuditFilter || '';
  const filtered   = filterUser ? entries.filter(e=>e.userId===filterUser) : entries;
  const users      = [...new Set(entries.map(e=>e.userId))].sort();
  const rows = filtered.slice(0,100).map(e =>
    '<tr style="border-bottom:1px solid #f1f5f9">'
    + '<td style="padding:7px 12px;font-size:11px;color:#94a3b8;white-space:nowrap">' + new Date(e.ts).toLocaleString() + '</td>'
    + '<td style="padding:7px 12px;font-size:12px;color:#374151;font-weight:600">' + escHtml(e.userId) + '</td>'
    + '<td style="padding:7px 12px;font-size:12px;color:#374151">' + escHtml(e.action) + '</td>'
    + '<td style="padding:7px 12px;font-size:12px;color:#64748b">' + escHtml(e.detail) + '</td>'
    + '</tr>'
  ).join('');
  el.innerHTML = '<div style="background:#fff;border:1px solid #e0e0e0;border-radius:8px;overflow:hidden">'
    + '<div style="padding:14px 16px;border-bottom:1px solid #e0e0e0;display:flex;align-items:center;gap:12px">'
    + '<span style="font-size:13px;font-weight:600;color:#374151">Audit Log</span>'
    + '<span style="font-size:12px;color:#94a3b8">' + filtered.length + ' events' + (filterUser?' for '+escHtml(filterUser):'') + '</span>'
    + '<select onchange="state.adminAuditFilter=this.value;renderAdmin(document.getElementById(\'main\'))" style="margin-left:auto;padding:5px 8px;border:1px solid #d1d5db;border-radius:6px;font-size:12px">'
    + '<option value="">All users</option>'
    + users.map(u=>'<option value="' + escHtml(u) + '"' + (u===filterUser?' selected':'') + '>' + escHtml(u) + '</option>').join('')
    + '</select></div>'
    + '<div style="overflow-x:auto;max-height:500px;overflow-y:auto">'
    + '<table style="width:100%;border-collapse:collapse">'
    + '<thead style="background:#f8fafc;position:sticky;top:0"><tr>'
    + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase;white-space:nowrap">Timestamp</th>'
    + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">User</th>'
    + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">Action</th>'
    + '<th style="padding:8px 12px;text-align:left;font-size:11px;color:#64748b;font-weight:700;text-transform:uppercase">Detail</th>'
    + '</tr></thead>'
    + '<tbody>' + (rows||'<tr><td colspan="4" style="padding:20px;text-align:center;color:#94a3b8">No events yet</td></tr>') + '</tbody>'
    + '</table></div></div>';
}
```

- [ ] **Step 3: Verify**

Admin → Commentary: select Mar 26 + professional_services, type a note, Save. Log in as swalski → amber callout visible.

Admin → Audit Log: shows login events from earlier in the session. Filter by user works.

- [ ] **Step 4: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: admin commentary editor and audit log viewer"
```

---

## Task 11: Admin save flow (File System Access API + download fallback)

**Files:**
- Modify: `monthly financial analysis.html` — add `saveToFile`, `buildUpdatedHTML`, `showSaveToast`

- [ ] **Step 1: Add save flow functions**

Insert after `renderAdminAudit`:
```js
// ═══════════════════════════════════════════════════════════════
// ADMIN SAVE FLOW
// ═══════════════════════════════════════════════════════════════

function serializeUsersBlock(){
  return '// @@USERS_START@@\nconst USERS = ' + JSON.stringify(USERS, null, 2) + ';\n// @@USERS_END@@';
}

function serializeCommentaryBlock(){
  return '// @@COMMENTARY_START@@\nconst COMMENTARY = ' + JSON.stringify(COMMENTARY, null, 2) + ';\n// @@COMMENTARY_END@@';
}

async function buildUpdatedHTML(){
  let html = document.documentElement.outerHTML;
  html = html.replace(/\/\/ @@USERS_START@@[\s\S]*?\/\/ @@USERS_END@@/, serializeUsersBlock());
  html = html.replace(/\/\/ @@COMMENTARY_START@@[\s\S]*?\/\/ @@COMMENTARY_END@@/, serializeCommentaryBlock());
  return html;
}

async function saveToFile(){
  const html = await buildUpdatedHTML();
  const blob = new Blob([html], {type:'text/html'});

  if('showSaveFilePicker' in window){
    try {
      const fh = await window.showSaveFilePicker({
        suggestedName: 'monthly financial analysis.html',
        types:[{description:'HTML file', accept:{'text/html':['.html']}}]
      });
      const w = await fh.createWritable();
      await w.write(blob);
      await w.close();
      showSaveToast('Saved to file.');
      return;
    } catch(e){
      if(e.name === 'AbortError') return;
    }
  }

  // Fallback: trigger download
  const url = URL.createObjectURL(blob);
  const a   = document.createElement('a');
  a.href    = url;
  a.download = 'monthly financial analysis.html';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
  showSaveToast('Downloaded. Replace the file in your project folder and let the auto-push sync it to GitHub.');
}

function showSaveToast(msg){
  let t = document.getElementById('save-toast');
  if(!t){
    t = document.createElement('div');
    t.id = 'save-toast';
    t.style.cssText = 'position:fixed;bottom:24px;right:24px;background:#1e40af;color:#fff;padding:12px 20px;border-radius:8px;font-size:13px;font-weight:600;z-index:9999;box-shadow:0 4px 12px rgba(0,0,0,0.2);display:none';
    document.body.appendChild(t);
  }
  t.textContent = msg;
  t.style.display = 'block';
  setTimeout(()=>{ t.style.display='none'; }, 4000);
}
```

- [ ] **Step 2: Verify sentinel regex**

```js
const html = document.documentElement.outerHTML;
/\/\/ @@USERS_START@@[\s\S]*?\/\/ @@USERS_END@@/.test(html)          // true
/\/\/ @@COMMENTARY_START@@[\s\S]*?\/\/ @@COMMENTARY_END@@/.test(html) // true
```

- [ ] **Step 3: Set initial passwords for all 8 accounts**

Open the file in Chrome as a local file (`file:///`). Open DevTools console.

Set admin passwords (run for each of scott, david, james):
```js
async function setPass(userId, password){
  const u = USERS.find(u=>u.id===userId);
  u.salt = generateSalt();
  u.passwordHash = await sha256WithSalt(u.salt, password);
  console.log(userId, 'hash set');
}
// Run for each admin:
await setPass('scott', 'ChooseAStrongPassword');
await setPass('david', 'ChooseAStrongPassword');
await setPass('james', 'ChooseAStrongPassword');
// Then trigger save:
await saveToFile();
```

Replace the file with the downloaded version. Reload. Log in as `scott` with the chosen password. Should succeed and see full admin dashboard.

Now use the Admin panel to set dept head passwords:
- Admin → Users → Stephen Walski → New Password field → type password → Save Changes
- Repeat for brandon, timc, nathan, dan

Each Save triggers `saveToFile()`. Replace the file after each (or save all five then save once).

- [ ] **Step 4: Verify all logins**

Test each of the 8 accounts. Confirm:
- Admins: full dashboard + Admin tab
- Dept heads: blue shell, scoped to their depts, no EBITDA

- [ ] **Step 5: Commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: admin save flow — FSA write + download fallback with sentinel replacement"
```

---

## Task 12: Final integration test

- [ ] **Auth — brute-force lockout**
  - Log in with correct username, wrong password 5 times → "Account locked for 15 minutes" message
  - Clear `sessionStorage` key `dp_lockout_swalski` in DevTools → can try again

- [ ] **Auth — session expiry**
  - Log in as dept head. In console: `const tok = JSON.parse(sessionStorage.getItem('dp_session')); tok.lastActive -= 4000000; sessionStorage.setItem('dp_session', JSON.stringify(tok));`
  - Wait 60 seconds (the interval check) → app locks automatically

- [ ] **Admin panel — full workflow**
  - Add a new user → set username + password + depts + views → Save → download/write → reload → new user can log in
  - Add commentary for "Mar 26:security" → log in as swalski → amber callout visible on Overview
  - Audit log shows: login_success, navigate, user_saved, commentary_saved events

- [ ] **Dept head data isolation**
  - Log in as `brandon` (Managed Services only) — no Security, PS, or other dept data anywhere
  - Log in as `swalski` — no EBITDA, no company revenue total, no other dept data
  - Intelligence tab: ask "what is total company EBITDA?" → advisor declines to answer

- [ ] **PDF export**
  - Click ⬇ Report → print dialog → preview shows blue header, KPI cards, no nav buttons

- [ ] **Mobile**
  - DevTools → 375px width → KPI cards single column, pills scroll, header wraps

- [ ] **Final commit**

```bash
git add "monthly financial analysis.html"
git commit -m "feat: RBAC complete — multi-user auth, admin panel, dept head views, scoped advisor"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|-----------------|------|
| Light/clean login screen | 2 |
| Username + password fields | 2 |
| Salted SHA-256 (per-user salt) | 1, 2 |
| Lockout: 5 attempts, 15 min | 2 |
| Dept heads: sessionStorage; Admins: localStorage | 2 |
| 1-hour inactivity timeout | 2 |
| Audit log (in-memory + localStorage, 500 events) | 1, 12 |
| Admin tab (admin only) | 3 |
| Logout button (all users) | 3, 5 |
| Master/detail user management | 9 |
| Dept/view pill toggles | 9 |
| Password reset per user | 9 |
| Add/remove user | 9 |
| Commentary editor (month + dept + text) | 10 |
| Audit log viewer with user filter | 10 |
| FSA save + download fallback | 11 |
| Sentinel-delimited USERS/COMMENTARY blocks | 1, 11 |
| Dept head blue shell (header, role badge) | 5 |
| Dept switcher pills | 5 |
| Month selector | 5 |
| "Data through" + PRELIMINARY indicator | 5 |
| KPI cards: Revenue, GP, Labor with colored borders | 6 |
| 6-month sparklines per KPI | 6 |
| Per-dept breakdown table | 6 |
| Commentary amber callout | 6 |
| Budget vs actual data model hook | 6 |
| P&L view (scoped) | 7 |
| GP view with % bars | 7 |
| Labor view | 7 |
| Sales view | 7 |
| Marketing view | 7 |
| Dept-scoped Claude advisor with guardrails | 7 |
| PDF export via window.print() | 8 |
| Mobile responsive styles | 8 |
| `escHtml()` on all user-supplied strings | 1, all innerHTML |
| Access matrix (8 users) | 1 |
| No company revenue/EBITDA for dept heads | 6, 7 |

**No gaps found.**
