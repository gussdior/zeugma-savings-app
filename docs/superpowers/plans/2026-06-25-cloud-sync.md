# Cloud Sync (Supabase) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automatically sync all app data (clients, prices, discounts) to a Supabase cloud database behind a shared spa login, local-first and offline-tolerant, with no bundled libraries.

**Architecture:** A single shared row (`spa_state.data` jsonb) holds the whole app-state bundle. The app talks to Supabase Auth + REST over plain `fetch()`. localStorage stays the working store; changes debounce-push to the cloud and the cloud is pulled on open (newer wins). RLS locks the row to authenticated users.

**Tech Stack:** Vanilla HTML/CSS/JS, `fetch`, Supabase (GoTrue Auth + PostgREST), localStorage. No dependencies, no build.

## Global Constraints

- Single offline file: **all code in `index.html`**. No libraries, no CDN, no build, no server we run.
- **No em dashes** in user-visible copy. Montserrat + existing palette vars.
- Reuse `BACKUP_KEYS` (the four keys `zeugma_services`, `zeugma_tiers`, `zeugma_invoice_seq`, `zeugma_clients`) defined in the profiles feature.
- **Cloud is optional until configured:** if `SUPA_URL` is still the placeholder, the app behaves exactly as today (no login, no sync). Login/sync only activate when real credentials are present (`CLOUD_ENABLED`).
- **Security (verbatim intent):** RLS enabled; anonymous denied; only `authenticated` may read/write; the anon public key in the page is safe only because of this. The shared password is entered per device and stored only as a session token, never embedded in the code.
- Supabase changes often: at build time, confirm the Auth/REST request shapes against current docs (`search_docs` / fetch `…​.md`) before finalizing; the shapes below follow stable GoTrue/PostgREST conventions.
- No test framework: verify pure/orchestration logic with `fetch` stubbed (Node/browser); verify auth+REST against the real project after Task 1.

---

### Task 1: Owner sets up the Supabase project (manual, produces credentials)

**Deliverable:** a live Supabase project with the `spa_state` table + RLS, one shared-login user, and the two values `SUPA_URL` and `SUPA_ANON_KEY` handed back.

This task is performed by the owner (guided). It is a prerequisite for verifying Tasks 2-6 end to end. The code tasks can be written and stub-verified before it completes, but cannot be deployed working until these values exist.

- [ ] **Step 1: Create account + project**
  Go to supabase.com, sign up (free), create a New Project. Choose a strong database password and a nearby region. Wait for it to finish provisioning.

- [ ] **Step 2: Create the table, RLS, and policies**
  Open the project's **SQL Editor**, paste this exactly, and click Run:

```sql
create table if not exists public.spa_state (
  id text primary key,
  data jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

-- Lock the table: no access unless a policy allows it.
alter table public.spa_state enable row level security;

-- Only signed-in users may touch it (anon role is intentionally NOT granted).
grant select, insert, update on public.spa_state to authenticated;

-- One shared record, one shared login: all authenticated access is intended.
-- Anonymous users have no policy and are therefore denied.
create policy "spa read"   on public.spa_state for select to authenticated using (true);
create policy "spa insert" on public.spa_state for insert to authenticated with check (true);
create policy "spa update" on public.spa_state for update to authenticated using (true) with check (true);
```

  (Rationale for `using (true)`: this is the rare correct case for it. There is exactly one shared row and one shared login, so there is no other user's data to protect from. Anonymous is still fully denied because no anon policy exists.)

- [ ] **Step 3: Create the shared spa login**
  In **Authentication -> Users -> Add user**, create one user with the spa email and a strong password, and enable **Auto Confirm** (so it can sign in immediately). This is the only login.

- [ ] **Step 4: Collect the two values**
  In **Settings -> API**, copy the **Project URL** (looks like `https://abcd.supabase.co`) and the **anon public** key (a long `eyJ...` string). Hand both back. These get baked into `index.html` in Task 2.

- [ ] **Step 5: (Implementer) Sanity-check anon is denied**
  Once the values exist, confirm RLS denies anonymous reads (should return `[]` or 401, never the data):

```bash
curl -s "$SUPA_URL/rest/v1/spa_state?select=*" -H "apikey: $SUPA_ANON_KEY" | head -c 200; echo
```
  Expected: empty array `[]` or a permission error, NOT any row data.

---

### Task 2: Cloud config + auth layer

**Files:** Modify `index.html` — add after the `BACKUP_KEYS` block (near the backup/restore code).

**Interfaces:**
- Produces: `SUPA_URL`, `SUPA_ANON`, `CLOUD_ENABLED`, `cloudTokens()`, `setCloudTokens(t)`, `cloudSession()`, `cloudLogin(email,password)->{ok,error?}`, `cloudLogout()`, `cloudRefresh()->bool`.

- [ ] **Step 1: Add config + auth functions**

```js
// ---- CLOUD CONFIG (filled in after Supabase setup) ----
const SUPA_URL = "REPLACE_WITH_PROJECT_URL";      // e.g. https://abcd.supabase.co
const SUPA_ANON = "REPLACE_WITH_ANON_PUBLIC_KEY"; // long eyJ... string
const CLOUD_ENABLED = /^https:\/\/.+\.supabase\.co/.test(SUPA_URL);
const SUPA_TOKENS_KEY = "zeugma_supa_tokens";
const SYNCED_AT_KEY = "zeugma_synced_at";

function cloudTokens(){ try{ return JSON.parse(localStorage.getItem(SUPA_TOKENS_KEY)||"null"); }catch(e){ return null; } }
function setCloudTokens(t){ if(t) localStorage.setItem(SUPA_TOKENS_KEY, JSON.stringify(t)); else localStorage.removeItem(SUPA_TOKENS_KEY); }
function cloudSession(){ const t=cloudTokens(); return (t && t.access_token) ? t : null; }
async function cloudLogin(email, password){
  try{
    const r = await fetch(SUPA_URL+"/auth/v1/token?grant_type=password", {
      method:"POST", headers:{ "apikey":SUPA_ANON, "Content-Type":"application/json" },
      body: JSON.stringify({ email:(email||"").trim(), password:password||"" })
    });
    if(!r.ok){ const e=await r.json().catch(()=>({})); return { ok:false, error:e.error_description||e.msg||"Sign in failed" }; }
    setCloudTokens(await r.json()); return { ok:true };
  }catch(e){ return { ok:false, error:"No connection. Check the internet and try again." }; }
}
function cloudLogout(){ setCloudTokens(null); }
async function cloudRefresh(){
  const t=cloudTokens(); if(!t || !t.refresh_token) return false;
  try{
    const r = await fetch(SUPA_URL+"/auth/v1/token?grant_type=refresh_token", {
      method:"POST", headers:{ "apikey":SUPA_ANON, "Content-Type":"application/json" },
      body: JSON.stringify({ refresh_token:t.refresh_token })
    });
    if(!r.ok) return false; setCloudTokens(await r.json()); return true;
  }catch(e){ return false; }
}
```

- [ ] **Step 2: Stub-verify token + login handling (browser console, fetch stubbed)**

```js
const real=window.fetch;
window.fetch=async(u,o)=>({ ok:true, json:async()=>({access_token:"a1",refresh_token:"r1",expires_in:3600}) });
setCloudTokens(null);
console.assert(cloudSession()===null,"no session initially");
await cloudLogin("x@y.com","pw");
console.assert(cloudSession() && cloudSession().access_token==="a1","login stored token");
cloudLogout();
console.assert(cloudSession()===null,"logout cleared token");
window.fetch=real; console.log("Task 2 OK");
```
Expected: no assertion errors.

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "Add Supabase cloud config and auth layer (fetch-based)"
```

---

### Task 3: Cloud data layer + state bundle

**Files:** Modify `index.html` — after Task 2's block.

**Interfaces:**
- Consumes: `cloudSession`, `cloudRefresh`, `SUPA_URL`, `SUPA_ANON`, `BACKUP_KEYS`.
- Produces: `appBundle()->obj`, `applyBundle(obj)`, `cloudFetch(path,opts,retry)->Response`, `cloudPull()->{data,updated_at}|null`, `cloudPush(bundle)->true`.

- [ ] **Step 1: Add bundle + REST functions**

```js
function appBundle(){ const o={}; BACKUP_KEYS.forEach(k=>{ const v=localStorage.getItem(k); if(v!=null) o[k]=v; }); return o; }
function applyBundle(obj){ if(!obj) return; BACKUP_KEYS.forEach(k=>{ if(obj[k]!=null) localStorage.setItem(k, obj[k]); }); }

async function cloudFetch(path, opts, retry){
  const t=cloudSession(); if(!t) throw new Error("not signed in");
  opts=opts||{};
  opts.headers=Object.assign({ "apikey":SUPA_ANON, "Authorization":"Bearer "+t.access_token }, opts.headers||{});
  const r=await fetch(SUPA_URL+"/rest/v1/"+path, opts);
  if(r.status===401 && !retry){ if(await cloudRefresh()) return cloudFetch(path, opts, true); }
  return r;
}
async function cloudPull(){
  const r=await cloudFetch("spa_state?id=eq.main&select=data,updated_at", { headers:{ "Accept":"application/json" } });
  if(!r.ok) throw new Error("pull "+r.status);
  const rows=await r.json(); return rows[0] || null;
}
async function cloudPush(bundle){
  const r=await cloudFetch("spa_state?on_conflict=id", {
    method:"POST",
    headers:{ "Content-Type":"application/json", "Prefer":"resolution=merge-duplicates,return=minimal" },
    body: JSON.stringify([{ id:"main", data:bundle, updated_at:new Date().toISOString() }])
  });
  if(!r.ok) throw new Error("push "+r.status);
  return true;
}
```

- [ ] **Step 2: Stub-verify bundle round-trip + 401 refresh-retry**

```js
// bundle round-trip
const snap=appBundle(); applyBundle(snap);
console.assert(typeof snap==="object","appBundle returns object");
// 401 -> refresh -> retry path
setCloudTokens({access_token:"old",refresh_token:"r"});
let calls=0; const real=window.fetch;
window.fetch=async(u,o)=>{
  if(u.includes("grant_type=refresh_token")) return { ok:true, json:async()=>({access_token:"new",refresh_token:"r2"}) };
  calls++; return { status: calls===1?401:200, ok: calls!==1, json:async()=>([{data:{},updated_at:"t"}]) };
};
const row=await cloudPull();
console.assert(calls===2 && cloudSession().access_token==="new","401 triggered refresh + retry");
window.fetch=real; setCloudTokens(null); console.log("Task 3 OK");
```
Expected: no assertion errors.

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "Add Supabase data layer (pull/push) and state bundle helpers"
```

---

### Task 4: Login screen + gate

**Files:** Modify `index.html` — add `#login` screen markup (after the rotate prompt), CSS before `</style>`, JS after Task 3, and gate startup.

**Interfaces:**
- Consumes: `CLOUD_ENABLED`, `cloudSession`, `cloudLogin`, `showScreen`, `pullOnOpen` (Task 5).
- Produces: `requireLoginIfNeeded()`, `doLogin()`.

- [ ] **Step 1: Add the login markup** (after the `#rotate` block):

```html
<!-- ============ LOGIN ============ -->
<div class="screen" id="login">
  <div class="login-card">
    <div class="login-brand">ZEUGMA&nbsp;MEDSPA</div>
    <div class="login-sub">Front desk sign in</div>
    <div class="field"><label>Email</label><input type="email" id="loginEmail" inputmode="email" autocomplete="username"></div>
    <div class="field"><label>Password</label><input type="password" id="loginPw" autocomplete="current-password"></div>
    <div class="err" id="loginErr"></div>
    <button class="btn btn-primary" id="loginBtn" type="button" style="width:100%; margin-top:6px;">Sign in</button>
    <div class="login-note">Your client list is private and only visible after you sign in.</div>
  </div>
</div>
```

- [ ] **Step 2: Add CSS** before `</style>`:

```css
#login{align-items:center; justify-content:center; background:linear-gradient(135deg,#6f6442 0%,#4A3C0E 70%);}
.login-card{background:#fff; border-radius:20px; padding:34px 32px; width:360px; max-width:92%; box-shadow:0 30px 70px -25px rgba(0,0,0,.55);}
.login-brand{font-size:20px; font-weight:400; letter-spacing:6px; color:var(--brown); text-align:center;}
.login-sub{font-size:12px; letter-spacing:2px; text-transform:uppercase; color:var(--muted); text-align:center; margin:6px 0 22px;}
.login-note{font-size:12px; color:var(--muted); text-align:center; margin-top:16px; line-height:1.5;}
```

- [ ] **Step 3: Add the gate + handler JS**

```js
function requireLoginIfNeeded(){
  if(CLOUD_ENABLED && !cloudSession()){ showScreen("login"); return true; }
  return false;
}
async function doLogin(){
  const email=document.getElementById("loginEmail").value;
  const pw=document.getElementById("loginPw").value;
  const err=document.getElementById("loginErr");
  const btn=document.getElementById("loginBtn");
  err.textContent=""; btn.disabled=true; btn.textContent="Signing in...";
  const res=await cloudLogin(email, pw);
  btn.disabled=false; btn.textContent="Sign in";
  if(!res.ok){ err.textContent=res.error; return; }
  document.getElementById("loginPw").value="";
  buildDashboard(); showScreen("dash");
  if(typeof pullOnOpen==="function") pullOnOpen();
}
document.getElementById("loginBtn").addEventListener("click", doLogin);
document.getElementById("loginPw").addEventListener("keydown", e=>{ if(e.key==="Enter") doLogin(); });
```

- [ ] **Step 4: Gate at startup.** Find the final `buildDashboard();` call at the end of the script and wrap it so a configured-but-logged-out app shows login instead:

```js
if(!requireLoginIfNeeded()){ buildDashboard(); }
```
(Replace the bare `buildDashboard();` end-of-script call with the line above. When cloud is not configured, `requireLoginIfNeeded()` returns false and the app builds the dashboard exactly as today.)

- [ ] **Step 5: Browser-verify (stubbed)**
  Temporarily set `CLOUD_ENABLED` true via console and confirm: with no session, `requireLoginIfNeeded()` shows `#login`; stub `cloudLogin` to succeed, click Sign in, and the dashboard appears. With cloud not configured (placeholder URL), the app loads straight to the dashboard (no login).

- [ ] **Step 6: Commit**

```bash
git add index.html && git commit -m "Add front-desk login screen and startup gate"
```

---

### Task 5: Sync orchestration (pull on open, debounced push, offline retry)

**Files:** Modify `index.html` — add after Task 3/4; add `markDirty()` calls into the existing save functions.

**Interfaces:**
- Consumes: `cloudSession`, `cloudPull`, `cloudPush`, `appBundle`, `applyBundle`, `SYNCED_AT_KEY`, `loadServices`, `loadTiers`, `buildDashboard`, `setSyncStatus` (Task 6).
- Produces: `markDirty()`, `pushNow()`, `pullOnOpen()`.

- [ ] **Step 1: Add orchestration JS**

```js
let syncTimer=null, syncing=false;
function syncStatus(s){ if(typeof setSyncStatus==="function") setSyncStatus(s); }
function markDirty(){
  if(!CLOUD_ENABLED || !cloudSession()) return;
  clearTimeout(syncTimer);
  syncStatus("saving");
  syncTimer=setTimeout(pushNow, 1500);
}
async function pushNow(){
  if(!CLOUD_ENABLED || !cloudSession() || syncing) return;
  if(!navigator.onLine){ syncStatus("offline"); return; }
  syncing=true;
  try{ await cloudPush(appBundle()); localStorage.setItem(SYNCED_AT_KEY, new Date().toISOString()); syncStatus("synced"); }
  catch(e){ syncStatus("offline"); }
  finally{ syncing=false; }
}
async function pullOnOpen(){
  if(!CLOUD_ENABLED || !cloudSession() || !navigator.onLine){ return; }
  try{
    const row=await cloudPull();
    if(row && row.data){
      const localAt=localStorage.getItem(SYNCED_AT_KEY)||"";
      if(!localAt || (row.updated_at && row.updated_at > localAt)){
        applyBundle(row.data);
        localStorage.setItem(SYNCED_AT_KEY, row.updated_at||new Date().toISOString());
        services=loadServices(); TIERS=loadTiers(); buildDashboard();
      }
    } else {
      await pushNow(); // first run: seed the cloud with current local data
    }
    syncStatus("synced");
  }catch(e){ syncStatus("offline"); }
}
window.addEventListener("online", pushNow);
document.addEventListener("visibilitychange", ()=>{ if(document.visibilityState==="visible") pullOnOpen(); });
```

- [ ] **Step 2: Hook `markDirty()` into every save**
  Append `markDirty();` inside these existing functions, right before they return / at their end:
  - `saveServices(svcs)` (after `localStorage.setItem(STORE_KEY, ...)`)
  - `saveTiers()` (after the setItem)
  - `saveClients(arr)` (after the setItem)
  - `nextInvoiceNo()` (after `localStorage.setItem(INV_KEY, ...)`)

  Example for `saveClients`:

```js
function saveClients(arr){ try{ localStorage.setItem(CLIENTS_KEY, JSON.stringify(arr)); }catch(e){} markDirty(); }
```
  (Apply the same one-line `markDirty();` addition to the other three.)

- [ ] **Step 3: Pull on successful login + startup**
  Confirm `doLogin()` (Task 4) calls `pullOnOpen()` after a successful sign in (it does). Also, at startup when already logged in, call `pullOnOpen()` once: in the end-of-script gate, extend to:

```js
if(!requireLoginIfNeeded()){ buildDashboard(); if(CLOUD_ENABLED && cloudSession()) pullOnOpen(); }
```

- [ ] **Step 4: Stub-verify orchestration**

```js
// debounce + push
setCloudTokens({access_token:"a",refresh_token:"r"});
let pushed=0; const real=window.fetch;
window.fetch=async(u,o)=>{ if(u.includes("/rest/")){ pushed++; return {ok:true,status:200,json:async()=>([])}; } return {ok:true,json:async()=>({})}; };
Object.defineProperty(navigator,'onLine',{value:true,configurable:true});
markDirty(); markDirty(); // two quick changes
await new Promise(r=>setTimeout(r,1700));
console.assert(pushed===1,"debounced to a single push, got "+pushed);
// offline: no push, status offline
Object.defineProperty(navigator,'onLine',{value:false,configurable:true});
pushed=0; await pushNow();
console.assert(pushed===0,"no push while offline");
Object.defineProperty(navigator,'onLine',{value:true,configurable:true});
window.fetch=real; setCloudTokens(null); console.log("Task 5 OK");
```
Expected: no assertion errors.

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "Add local-first sync orchestration (pull on open, debounced push, offline retry)"
```

---

### Task 6: Sync status chip + sign out

**Files:** Modify `index.html` — add a status chip to the dashboard header and a Sign out control on the Clients screen; CSS before `</style>`; JS for `setSyncStatus` and sign out.

**Interfaces:**
- Consumes: `cloudLogout`, `requireLoginIfNeeded`, `CLOUD_ENABLED`, `cloudSession`.
- Produces: `setSyncStatus(state)`.

- [ ] **Step 1: Add the chip to the dashboard header**
  In the dashboard header button group (where `clientsBtn`/`settingsBtn`/`addBtn` live), add as the first child:

```html
      <span class="sync-chip" id="syncChip" data-state="idle" style="display:none;">Synced</span>
```

- [ ] **Step 2: Add a Sign out button on the Clients screen header**
  In the `#clients` header button group (next to Back up / Restore / Done), add:

```html
      <button class="add-btn" id="signoutBtn" title="Sign out of this iPad" style="display:none;">Sign out</button>
```

- [ ] **Step 3: Add CSS** before `</style>`:

```css
.sync-chip{display:inline-flex; align-items:center; gap:6px; font-size:12px; font-weight:600; padding:9px 14px; border-radius:30px; border:1px solid #6b5a22; color:#efe7d2; background:rgba(255,255,255,.08);}
.sync-chip[data-state="saving"]{color:#e9dcb4;}
.sync-chip[data-state="offline"]{color:#e8b9a8; border-color:#7a5040;}
.sync-chip[data-state="synced"]{color:#cfe0bf;}
```

- [ ] **Step 4: Add JS** (after Task 5):

```js
const SYNC_TEXT={ idle:"", saving:"Saving...", synced:"Synced", offline:"Offline, will sync" };
function setSyncStatus(state){
  const el=document.getElementById("syncChip"); if(!el) return;
  el.dataset.state=state; el.textContent=SYNC_TEXT[state]||"";
  el.style.display = (CLOUD_ENABLED && cloudSession()) ? "inline-flex" : "none";
}
function refreshCloudControls(){
  const on = CLOUD_ENABLED && !!cloudSession();
  const so=document.getElementById("signoutBtn"); if(so) so.style.display = on ? "inline-flex" : "none";
  setSyncStatus(cloudSession()?"synced":"idle");
}
document.getElementById("signoutBtn").addEventListener("click", ()=>{
  if(!confirm("Sign out of this iPad? Your data stays safe in the cloud and on this device.")) return;
  cloudLogout(); requireLoginIfNeeded();
});
refreshCloudControls();
```

- [ ] **Step 5: Show the chip after login + on startup**
  Ensure `refreshCloudControls()` runs after a successful `doLogin()` and on startup when logged in. Add `refreshCloudControls();` at the end of `doLogin()` (after `showScreen("dash")`) and in the startup gate branch where `cloudSession()` is truthy.

- [ ] **Step 6: Browser-verify (stubbed)**
  With a stubbed session: chip shows "Synced"; trigger `markDirty()` -> "Saving..." then "Synced"; go offline + `pushNow()` -> "Offline, will sync"; Sign out -> login screen returns and the chip hides.

- [ ] **Step 7: Commit**

```bash
git add index.html && git commit -m "Add sync status chip and sign out"
```

---

### Task 7: Bake credentials + real end-to-end verification (after Task 1 delivered)

**Files:** Modify `index.html` — replace the two `REPLACE_WITH_...` placeholders with the owner's real `SUPA_URL` and `SUPA_ANON` from Task 1.

- [ ] **Step 1:** Put the real Project URL and anon key into the `SUPA_URL` / `SUPA_ANON` constants. Confirm `CLOUD_ENABLED` becomes true.

- [ ] **Step 2: Real end-to-end (served over http, real network)**
  1. Load the app -> the login screen appears. Sign in with the shared login -> dashboard.
  2. Send an invoice / mark a session -> chip shows "Saving..." then "Synced". In Supabase Table editor, `spa_state.data` contains the updated `zeugma_clients`.
  3. Open the app in a second browser/profile, sign in -> the same client + counts appear (pull on open).
  4. Go offline (DevTools offline), mark a session -> chip "Offline, will sync"; go online -> it pushes and shows "Synced"; confirm the change is in Supabase.
  5. Sign out -> login screen; an anonymous `curl` (Task 1 Step 5) still returns no data.

- [ ] **Step 3: Commit + deploy**

```bash
git add index.html && git commit -m "Wire live Supabase credentials and enable cloud sync"
```
  Then merge to `main` and push (GitHub Pages deploy), and verify the live URL serves the login screen.

---

## Self-Review

**Spec coverage:**
- Supabase project + `spa_state` jsonb + RLS authenticated-only, anon denied -> Task 1 (SQL + curl check). ✓
- Shared login, password per device as token -> Task 2 (auth), Task 4 (login UI). ✓
- All data as one JSON doc -> Task 3 (`appBundle`/`applyBundle` over `BACKUP_KEYS`). ✓
- Local-first, pull on open (newer wins), debounced push, offline retry -> Task 5. ✓
- No bundled library (plain fetch) -> Tasks 2-3. ✓
- Sync status visible + sign out -> Task 6. ✓
- Cloud optional until configured (`CLOUD_ENABLED`) -> Tasks 2,4,5,6. ✓
- Backup/Restore retained -> untouched (still present). ✓
- Live verification (cross-device, offline, anon-denied) -> Tasks 1.5 + 7. ✓
- No em dashes in new copy -> login/chip/confirm strings checked. ✓

**Placeholder scan:** The only intentional placeholders are the two credential constants (`REPLACE_WITH_...`), filled in Task 7 from Task 1's output — by design, not a plan gap. No TBD logic; every code step is complete.

**Type/name consistency:** `cloudSession`, `cloudLogin`, `cloudRefresh`, `cloudFetch`, `cloudPull`, `cloudPush`, `appBundle`, `applyBundle`, `markDirty`, `pushNow`, `pullOnOpen`, `setSyncStatus`, `requireLoginIfNeeded`, `refreshCloudControls`, and keys `SUPA_TOKENS_KEY` / `SYNCED_AT_KEY` are defined once and used consistently. `setSyncStatus` takes a state string from `{idle,saving,synced,offline}` used identically in Tasks 5 and 6 (Task 5 calls it via the `syncStatus()` shim that no-ops until Task 6 defines it).

**Cross-task note:** Task 5 references `setSyncStatus` (Task 6) through a `typeof`-guarded shim, and Task 4 references `pullOnOpen` (Task 5) through a `typeof` guard, so intermediate states are safe. Implement in order 2->3->4->5->6, then 7 once credentials arrive.
