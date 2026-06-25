# Client Profiles + Sessions-Left Email Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add client profiles (created when an invoice is sent) that track sessions left per treatment, let the front desk mark sessions used behind a confirm popup, hold a manually entered next-appointment date, send a warm one-tap sessions-update email, and back up/restore all app data.

**Architecture:** All in the single offline `index.html`. A new `localStorage` key `zeugma_clients` holds client records; pure helpers manage it and build a `mailto:` email. A new Clients screen + a profile modal + a confirm modal provide the UI. Profile creation hooks into the existing `makeInvoicePDF()`.

**Tech Stack:** Vanilla HTML/CSS/JS, no framework, no build, no dependencies; `localStorage`; `mailto:`; Blob download + file input for backup/restore.

## Global Constraints

- Single offline file: **all changes in `index.html`**. No new files, no libraries, no CDN, no build step, no server.
- **No em dashes** in any user-visible copy (brand rule). Use commas, periods, "to", or "-".
- Montserrat font; palette vars only (`--olive #4A3C0E`, cream `#E9E0CE`/`#EFEAE0`, `--coral #C0664A`, sage-ink `#51662f`, `--muted`, `--line`, `--brown`).
- Business constants exist: `BIZ = { web:"zeugmaspa.com", ig:"@zeugmamedspa", phone:"(305) 481-5659" }`.
- Reuse existing helpers: `esc()`, `usd()` (not needed here), `loadServices`/`saveServices` pattern, modal `.modal-back.show` reveal, `showScreen(id)`.
- Storage key: **`zeugma_clients`**. Client record shape:
  `{ id, name, email, phone, nextAppt: string|null, packages: [ { name, total, used } ] }`. Match key `id` = email lowercased+trimmed. `sessionsLeft = total - used`.
- Email is **one-tap `mailto:`** (plain text, no attachment). Subject: `A little update on your Zeugma treatments`.
- No test framework in this repo: verify pure functions with Node snippets, UI with Playwright/manual. Commit after each task.

---

### Task 1: Client store + model helpers (pure logic)

**Files:**
- Modify: `index.html` — add after `saveServices()` (`index.html:602`).

**Interfaces:**
- Produces:
  - `CLIENTS_KEY = "zeugma_clients"`
  - `loadClients()` -> array; `saveClients(arr)`
  - `clientIdFor(email)` -> lowercased trimmed string
  - `findClient(email)` -> client | undefined
  - `pkgLeft(p)` -> number
  - `clientActivePkgs(c)` -> packages with `pkgLeft > 0`
  - `upsertClientFromInvoice(client, items)` -> client. `client` is `{name,email,phone}`; `items` is the invoice items array `[{name, qty, ...}]`.

- [ ] **Step 1: Add the store + helpers**

Insert after `saveServices()` (`index.html:602`):

```js
const CLIENTS_KEY = "zeugma_clients";
function loadClients(){ try{ const a=JSON.parse(localStorage.getItem(CLIENTS_KEY)||"null"); if(Array.isArray(a)) return a; }catch(e){} return []; }
function saveClients(arr){ try{ localStorage.setItem(CLIENTS_KEY, JSON.stringify(arr)); }catch(e){} }
function clientIdFor(email){ return (email||"").trim().toLowerCase(); }
function findClient(email){ const id=clientIdFor(email); return loadClients().find(c=>c.id===id); }
function pkgLeft(p){ return Math.max(0, (p.total||0) - (p.used||0)); }
function clientActivePkgs(c){ return (c.packages||[]).filter(p=>pkgLeft(p)>0); }
function upsertClientFromInvoice(client, items){
  const id = clientIdFor(client.email);
  if(!id) return null;
  const clients = loadClients();
  let c = clients.find(x=>x.id===id);
  if(!c){ c={ id, name:client.name||"", email:client.email||"", phone:client.phone||"", nextAppt:null, packages:[] }; clients.push(c); }
  else { if(client.name) c.name=client.name; if(client.phone) c.phone=client.phone; }
  (items||[]).forEach(it=>{
    const active = c.packages.find(p=>p.name===it.name && pkgLeft(p)>0);
    if(active){ active.total += it.qty; }
    else { c.packages.push({ name:it.name, total:it.qty, used:0 }); }
  });
  saveClients(clients);
  return c;
}
```

- [ ] **Step 2: Node-verify the model**

```bash
cd "C:/Users/tipal/OneDrive/Desktop/dev/zeugma-savings" && node -e '
const store={}; global.localStorage={getItem:k=>k in store?store[k]:null,setItem:(k,v)=>store[k]=String(v),removeItem:k=>delete store[k]};
const CLIENTS_KEY="zeugma_clients";
function loadClients(){try{const a=JSON.parse(localStorage.getItem(CLIENTS_KEY)||"null");if(Array.isArray(a))return a;}catch(e){}return[];}
function saveClients(arr){try{localStorage.setItem(CLIENTS_KEY,JSON.stringify(arr));}catch(e){}}
function clientIdFor(email){return(email||"").trim().toLowerCase();}
function pkgLeft(p){return Math.max(0,(p.total||0)-(p.used||0));}
function upsertClientFromInvoice(client,items){const id=clientIdFor(client.email);if(!id)return null;const clients=loadClients();let c=clients.find(x=>x.id===id);if(!c){c={id,name:client.name||"",email:client.email||"",phone:client.phone||"",nextAppt:null,packages:[]};clients.push(c);}else{if(client.name)c.name=client.name;if(client.phone)c.phone=client.phone;}(items||[]).forEach(it=>{const active=c.packages.find(p=>p.name===it.name&&pkgLeft(p)>0);if(active){active.total+=it.qty;}else{c.packages.push({name:it.name,total:it.qty,used:0});}});saveClients(clients);return c;}

upsertClientFromInvoice({name:"Jane Doe",email:"JANE@email.com ",phone:"555"},[{name:"Laser",qty:6}]);
let c=loadClients()[0];
console.assert(loadClients().length===1 && c.id==="jane@email.com","create+normalize id: "+c.id);
console.assert(c.packages[0].total===6 && c.packages[0].used===0,"pkg seeded");
// same email + same active treatment tops up
upsertClientFromInvoice({name:"Jane Doe",email:"jane@email.com"},[{name:"Laser",qty:6}]);
c=loadClients()[0];
console.assert(loadClients().length===1,"no duplicate client");
console.assert(c.packages.length===1 && c.packages[0].total===12,"top-up to 12: "+c.packages[0].total);
// different treatment adds a row
upsertClientFromInvoice({name:"Jane Doe",email:"jane@email.com"},[{name:"Hydrafacial",qty:4}]);
console.assert(loadClients()[0].packages.length===2,"second treatment row");
console.log("Task 1 OK", JSON.stringify(loadClients()[0].packages));
'
```
Expected: no assertion errors; logs the two packages.

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "Add client store and profile model helpers"
```

---

### Task 2: Create the profile when an invoice is sent

**Files:**
- Modify: `index.html` — `makeInvoicePDF()` (`index.html:1167`).

**Interfaces:**
- Consumes: `upsertClientFromInvoice(client, items)` (Task 1), `invoiceData()` (existing, returns `{client, items, totals}`).

- [ ] **Step 1: Hook profile creation into invoice send**

In `makeInvoicePDF()`, after `const d = invoiceData();` and before `buildInvoice(...)` (`index.html:1175-1176`), add the upsert:

```js
  const d = invoiceData();
  upsertClientFromInvoice(d.client, d.items);
  buildInvoice({ invoiceNo:invoiceCtx.invoiceNo, dateStr:invoiceDateStr(), client:d.client, items:d.items, totals:d.totals });
```

- [ ] **Step 2: Browser-verify (Playwright, served over http)**

Serve the folder, open it, stub print, send a single-service invoice, and confirm a client was stored:

```js
// in page context
localStorage.removeItem('zeugma_clients');
window.print=()=>{};
const key=document.querySelector('[data-invoice]').dataset.invoice;
openInvoiceForm({type:'service', key});
document.getElementById('invName').value='Test Client';
document.getElementById('invEmail').value='test@x.com';
makeInvoicePDF();
const c=loadClients().find(x=>x.id==='test@x.com');
return { stored: !!c, pkgs: c && c.packages, name: c && c.name };
```
Expected: `stored:true`, one package with `total` = the chosen sessions, `used:0`.

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "Create/update client profile when an invoice is sent"
```

---

### Task 3: Sessions-update email builder (pure logic)

**Files:**
- Modify: `index.html` — add near the client helpers (after Task 1's block).

**Interfaces:**
- Consumes: `clientActivePkgs(c)`, `pkgLeft(p)`, `BIZ`.
- Produces:
  - `formatApptDate(iso)` -> "Saturday, July 8, 2026" or "" for falsy input
  - `clientUpdateBody(c)` -> plain-text email body (string)
  - `clientUpdateMailto(c)` -> `mailto:` URL string

- [ ] **Step 1: Add the email builder**

```js
const WD = ["Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"];
const MO = ["January","February","March","April","May","June","July","August","September","October","November","December"];
function formatApptDate(iso){
  if(!iso) return "";
  const p=String(iso).split("-"); if(p.length!==3) return "";
  const d=new Date(+p[0], +p[1]-1, +p[2]);
  if(isNaN(d)) return "";
  return WD[d.getDay()]+", "+MO[d.getMonth()]+" "+d.getDate()+", "+d.getFullYear();
}
function clientUpdateBody(c){
  const first=(c.name||"there").trim().split(/\s+/)[0];
  const lines=[];
  lines.push("Hi "+first+",");
  lines.push("");
  lines.push("We hope you're feeling wonderful. Here is a friendly check-in on your treatment packages with us:");
  lines.push("");
  clientActivePkgs(c).forEach(p=>{ lines.push("- "+p.name+": "+pkgLeft(p)+" of "+p.total+" sessions left"); });
  lines.push("");
  const appt=formatApptDate(c.nextAppt);
  if(appt){ lines.push("We have you booked for your next visit on "+appt+", and we cannot wait to see you."); lines.push(""); }
  lines.push("If you have any questions, or you would like to move things around, just give us a call. We are always happy to help.");
  lines.push("");
  lines.push("Warmly,");
  lines.push("The Zeugma MedSpa team");
  lines.push(BIZ.phone+" · "+BIZ.web);
  return lines.join("\n");
}
function clientUpdateMailto(c){
  const subject="A little update on your Zeugma treatments";
  return "mailto:"+(c.email||"")+"?subject="+encodeURIComponent(subject)+"&body="+encodeURIComponent(clientUpdateBody(c));
}
```

- [ ] **Step 2: Node-verify the email**

```bash
cd "C:/Users/tipal/OneDrive/Desktop/dev/zeugma-savings" && node -e '
const BIZ={web:"zeugmaspa.com",ig:"@zeugmamedspa",phone:"(305) 481-5659"};
const WD=["Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"];
const MO=["January","February","March","April","May","June","July","August","September","October","November","December"];
function pkgLeft(p){return Math.max(0,(p.total||0)-(p.used||0));}
function clientActivePkgs(c){return(c.packages||[]).filter(p=>pkgLeft(p)>0);}
function formatApptDate(iso){if(!iso)return"";const p=String(iso).split("-");if(p.length!==3)return"";const d=new Date(+p[0],+p[1]-1,+p[2]);if(isNaN(d))return"";return WD[d.getDay()]+", "+MO[d.getMonth()]+" "+d.getDate()+", "+d.getFullYear();}
function clientUpdateBody(c){const first=(c.name||"there").trim().split(/\s+/)[0];const lines=[];lines.push("Hi "+first+",");lines.push("");lines.push("We hope you'\''re feeling wonderful. Here is a friendly check-in on your treatment packages with us:");lines.push("");clientActivePkgs(c).forEach(p=>{lines.push("- "+p.name+": "+pkgLeft(p)+" of "+p.total+" sessions left");});lines.push("");const appt=formatApptDate(c.nextAppt);if(appt){lines.push("We have you booked for your next visit on "+appt+", and we cannot wait to see you.");lines.push("");}lines.push("If you have any questions, or you would like to move things around, just give us a call. We are always happy to help.");lines.push("");lines.push("Warmly,");lines.push("The Zeugma MedSpa team");lines.push(BIZ.phone+" · "+BIZ.web);return lines.join("\n");}

const c={name:"Jane Doe",email:"jane@x.com",nextAppt:"2026-07-08",packages:[{name:"Laser",total:6,used:3},{name:"Hydrafacial",total:4,used:2},{name:"Done",total:2,used:2}]};
const b=clientUpdateBody(c);
console.assert(formatApptDate("2026-07-08")==="Wednesday, July 8, 2026",formatApptDate("2026-07-08"));
console.assert(b.includes("Laser: 3 of 6 sessions left"),"laser line");
console.assert(b.includes("Hydrafacial: 2 of 4 sessions left"),"hydra line");
console.assert(!b.includes("Done:"),"completed package excluded");
console.assert(b.includes("July 8, 2026"),"appt present");
console.assert(b.indexOf("—")===-1,"no em dash");
const c2={name:"Bob",email:"b@x.com",nextAppt:null,packages:[{name:"Laser",total:6,used:0}]};
console.assert(clientUpdateBody(c2).indexOf("next visit")===-1,"no appt line when unset");
console.log("Task 3 OK");
'
```
Expected: no assertion errors; logs `Task 3 OK`. (Note: 2026-07-08 is a Wednesday; the mockup's "Saturday" was sample text only. The function computes the real weekday.)

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "Add warm sessions-update email builder (mailto, no em dashes)"
```

---

### Task 4: Clients screen (markup, CSS, navigation, list + search)

**Files:**
- Modify: `index.html` — dash header (`index.html:333-335` area, the `<div style="display:flex; gap:10px;">`), add a `#clients` screen after the `#dash` screen closes, add CSS before `</style>`, add JS near the other screen logic.

**Interfaces:**
- Consumes: `loadClients`, `clientActivePkgs`, `pkgLeft`, `esc`, `showScreen`, `buildDashboard`.
- Produces: `openClients()`, `buildClientsList(filter)`, `openClientProfile(id)` (defined in Task 5; in this task wire the card click to call it).

- [ ] **Step 1: Add a "Clients" button to the dashboard header**

Replace the header button group (the `<div style="display:flex; gap:10px;">` holding `settingsBtn` and `addBtn`) so it also has a Clients button first:

```html
    <div style="display:flex; gap:10px;">
      <button class="add-btn" id="clientsBtn" title="Client profiles and sessions left">
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M23 21v-2a4 4 0 0 0-3-3.87"/></svg>
        Clients
      </button>
      <button class="add-btn" id="settingsBtn" title="Edit package discount percentages">
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="19" y1="5" x2="5" y2="19"/><circle cx="6.5" cy="6.5" r="2.5"/><circle cx="17.5" cy="17.5" r="2.5"/></svg>
        Discounts
      </button>
      <button class="add-btn" id="addBtn">
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>
        Add service
      </button>
    </div>
```

- [ ] **Step 2: Add the Clients screen markup**

Immediately after the `#dash` screen's closing `</div>` (the dashboard screen, which ends just before `<!-- ============ CLIENT SAVINGS ============ -->`), insert:

```html
<!-- ============ CLIENTS ============ -->
<div class="screen" id="clients">
  <div class="dash-head">
    <div class="brand"><div class="b">CLIENTS</div><div class="s">SESSIONS &amp; FOLLOW-UP</div></div>
    <div style="display:flex; gap:10px;">
      <button class="add-btn" id="backupBtn" title="Save a backup file of all your data">Back up</button>
      <button class="add-btn" id="restoreBtn" title="Restore from a backup file">Restore</button>
      <button class="add-btn" id="clientsBack">Done</button>
    </div>
  </div>
  <div class="dash-body">
    <div class="cl-search">
      <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="11" cy="11" r="7"/><line x1="21" y1="21" x2="16.5" y2="16.5"/></svg>
      <input type="text" id="clSearch" placeholder="Search client by name">
    </div>
    <div id="clList"></div>
    <input type="file" id="restoreFile" accept="application/json" style="display:none">
  </div>
</div>
```

- [ ] **Step 3: Add CSS** before `</style>`:

```css
/* ---------- CLIENTS ---------- */
.cl-search{display:flex; align-items:center; gap:9px; background:#fff; border:1px solid #d8cdb6; border-radius:11px; padding:10px 14px; margin-bottom:18px; color:var(--muted); max-width:520px;}
.cl-search input{border:none; background:transparent; font-family:inherit; font-size:15px; color:var(--brown); width:100%; outline:none;}
.cl-card{background:#fff; border:1px solid var(--line); border-radius:16px; padding:16px 20px; margin-bottom:12px; display:flex; align-items:center; gap:16px; cursor:pointer; box-shadow:var(--shadow-sm); transition:transform .12s;}
.cl-card:active{transform:scale(.995);}
.cl-card .cl-nm{flex:1; min-width:0;}
.cl-card .cl-nm .n{font-size:18px; font-weight:600; color:var(--brown);}
.cl-card .cl-nm .e{font-size:12px; color:var(--muted); margin-top:2px;}
.cl-card .cl-meta{text-align:right; flex-shrink:0;}
.cl-card .cl-meta .big{font-size:20px; font-weight:800; color:var(--coral); line-height:1;}
.cl-card .cl-meta .lbl{font-size:11px; color:var(--muted); margin-top:2px;}
.cl-empty{color:var(--muted); font-size:14px; padding:20px 2px;}
```

- [ ] **Step 4: Add the list + search + nav JS** (near the other screen handlers, e.g. after the bundle handlers):

```js
function openClients(){ buildClientsList(""); document.getElementById("clSearch").value=""; showScreen("clients"); }
function buildClientsList(filter){
  const f=(filter||"").trim().toLowerCase();
  const list=loadClients().filter(c=>!f || (c.name||"").toLowerCase().includes(f));
  const wrap=document.getElementById("clList");
  if(!list.length){ wrap.innerHTML='<div class="cl-empty">No clients yet. Profiles are created when you send an invoice.</div>'; return; }
  wrap.innerHTML=list.map(c=>{
    const left=clientActivePkgs(c).reduce((a,p)=>a+pkgLeft(p),0);
    const n=clientActivePkgs(c).length;
    return `<div class="cl-card" data-cid="${esc(c.id)}">
      <div class="cl-nm"><div class="n">${esc(c.name||c.email)}</div><div class="e">${esc(c.email)}${c.phone?(" · "+esc(c.phone)):""}</div></div>
      <div class="cl-meta"><div class="big">${left}</div><div class="lbl">${left===1?"session":"sessions"} left · ${n} active</div></div>
    </div>`;
  }).join("");
  wrap.querySelectorAll(".cl-card").forEach(el=>el.addEventListener("click",()=>openClientProfile(el.dataset.cid)));
}
document.getElementById("clientsBtn").addEventListener("click", openClients);
document.getElementById("clientsBack").addEventListener("click", ()=>{ buildDashboard(); showScreen("dash"); });
document.getElementById("clSearch").addEventListener("input", e=>buildClientsList(e.target.value));
```

> Note: `openClientProfile` is defined in Task 5. Until Task 5 lands, clicking a card throws; that is expected mid-build and resolved by Task 5.

- [ ] **Step 5: Browser-verify**

Serve + open. In console: seed two clients via `upsertClientFromInvoice`, call `openClients()`, confirm two `.cl-card`s render, type in `#clSearch` to filter to one, and the "Done" button returns to the dashboard. (Card click is verified in Task 5.)

- [ ] **Step 6: Commit**

```bash
git add index.html && git commit -m "Add Clients screen with search and list"
```

---

### Task 5: Profile modal — packages, mark-used confirm, next appointment, email

**Files:**
- Modify: `index.html` — add `#clientBack` and `#sessConfirmBack` modals (after the invoice modal), CSS before `</style>`, JS after Task 4's block.

**Interfaces:**
- Consumes: `findClient` by id (use `loadClients().find(c=>c.id===id)`), `clientActivePkgs`, `pkgLeft`, `clientUpdateMailto`, `formatApptDate`, `saveClients`, `esc`, `buildClientsList`.
- Produces: `openClientProfile(id)`, `renderClientProfile()`, `markSessionUsed(idx)`, `setNextAppt(iso)`.

- [ ] **Step 1: Add the profile modal + confirm modal markup** after the invoice modal (`#invoiceBack`):

```html
<!-- ============ CLIENT PROFILE ============ -->
<div class="modal-back" id="clientBack">
  <div class="modal">
    <h2 id="clpName">Client</h2>
    <div class="sub" id="clpContact"></div>
    <div id="clpPkgs"></div>
    <div class="field" style="margin-top:6px;">
      <label>Next appointment (you enter this)</label>
      <input type="date" id="clpAppt">
      <div class="hint" id="clpApptShown"></div>
    </div>
    <div class="modal-actions">
      <button class="btn btn-ghost" id="clpClose" type="button">Close</button>
      <button class="btn btn-primary" id="clpEmail" type="button">Email sessions update</button>
    </div>
  </div>
</div>

<!-- ============ SESSION CONFIRM ============ -->
<div class="modal-back" id="sessConfirmBack" style="z-index:600;">
  <div class="modal" style="max-width:360px; text-align:center;">
    <h2>Has this session been done?</h2>
    <div class="sub" id="sessConfirmMsg" style="margin-bottom:22px;"></div>
    <div class="modal-actions" style="justify-content:center;">
      <button class="btn btn-ghost" id="sessConfirmNo" type="button">Not yet</button>
      <button class="btn btn-primary" id="sessConfirmYes" type="button">Yes, mark it used</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Add CSS** before `</style>`:

```css
.clp-pkg{padding:14px 0; border-bottom:1px solid var(--line);}
.clp-pkg:last-child{border-bottom:none;}
.clp-row{display:flex; align-items:center; gap:12px;}
.clp-row .clp-nm{flex:1; min-width:0;}
.clp-row .clp-nm .n{font-size:15px; font-weight:600; color:var(--brown);}
.clp-row .clp-nm .l{font-size:12px; font-weight:600; color:var(--sage-ink); margin-top:3px;}
.clp-row .clp-nm .l.done{color:var(--muted);}
.clp-bar{height:7px; background:#eadfca; border-radius:6px; overflow:hidden; margin-top:9px;}
.clp-bar > div{height:100%; background:var(--coral);}
.clp-use{font-family:inherit; font-size:12px; font-weight:600; color:var(--brown); background:var(--cream); border:1px solid var(--line); border-radius:20px; padding:8px 14px; cursor:pointer; white-space:nowrap;}
.clp-use:disabled{opacity:.45; cursor:default;}
```

- [ ] **Step 3: Add the profile JS** after Task 4's block:

```js
let clpId = null;
function currentClient(){ return loadClients().find(c=>c.id===clpId); }
function openClientProfile(id){
  clpId=id; const c=currentClient(); if(!c) return;
  renderClientProfile();
  document.getElementById("clientBack").classList.add("show");
}
function renderClientProfile(){
  const c=currentClient(); if(!c) return;
  document.getElementById("clpName").textContent=c.name||c.email;
  document.getElementById("clpContact").textContent=c.email+(c.phone?(" · "+c.phone):"");
  document.getElementById("clpPkgs").innerHTML=(c.packages||[]).map((p,i)=>{
    const left=pkgLeft(p), done=left===0, pct=p.total?Math.round((p.used/p.total)*100):0;
    return `<div class="clp-pkg">
      <div class="clp-row">
        <div class="clp-nm"><div class="n">${esc(p.name)}</div><div class="l ${done?"done":""}">${done?"Complete":(left+" of "+p.total+" sessions left")}</div></div>
        <button class="clp-use" data-use="${i}" ${done?"disabled":""}>&#8722; Mark session used</button>
      </div>
      <div class="clp-bar"><div style="width:${pct}%"></div></div>
    </div>`;
  }).join("");
  document.getElementById("clpPkgs").querySelectorAll("[data-use]").forEach(b=>b.addEventListener("click",()=>askSessionUsed(+b.dataset.use)));
  document.getElementById("clpAppt").value=c.nextAppt||"";
  document.getElementById("clpApptShown").textContent=c.nextAppt?formatApptDate(c.nextAppt):"No next appointment set.";
}
let pendingUseIdx=null;
function askSessionUsed(idx){
  const c=currentClient(); if(!c) return; const p=c.packages[idx]; if(!p||pkgLeft(p)<=0) return;
  pendingUseIdx=idx;
  document.getElementById("sessConfirmMsg").innerHTML="Mark one <b>"+esc(p.name)+"</b> session as used for <b>"+esc(c.name||c.email)+"</b>. Sessions left will go from <b>"+pkgLeft(p)+"</b> to <b>"+(pkgLeft(p)-1)+"</b>.";
  document.getElementById("sessConfirmBack").classList.add("show");
}
function confirmSessionUsed(){
  const c=currentClient(); const idx=pendingUseIdx;
  if(c && idx!=null && c.packages[idx] && pkgLeft(c.packages[idx])>0){
    const all=loadClients(); const cc=all.find(x=>x.id===clpId); cc.packages[idx].used++; saveClients(all);
    renderClientProfile();
  }
  pendingUseIdx=null;
  document.getElementById("sessConfirmBack").classList.remove("show");
}
function setNextAppt(iso){
  const all=loadClients(); const cc=all.find(x=>x.id===clpId); if(!cc) return;
  cc.nextAppt=iso||null; saveClients(all);
  document.getElementById("clpApptShown").textContent=iso?formatApptDate(iso):"No next appointment set.";
}
document.getElementById("clpClose").addEventListener("click",()=>{ document.getElementById("clientBack").classList.remove("show"); buildClientsList(document.getElementById("clSearch").value); });
document.getElementById("clientBack").addEventListener("click",e=>{ if(e.target.id==="clientBack"){ document.getElementById("clientBack").classList.remove("show"); buildClientsList(document.getElementById("clSearch").value); } });
document.getElementById("clpAppt").addEventListener("change",e=>setNextAppt(e.target.value));
document.getElementById("clpEmail").addEventListener("click",()=>{ const c=currentClient(); if(c) location.href=clientUpdateMailto(c); });
document.getElementById("sessConfirmNo").addEventListener("click",()=>{ pendingUseIdx=null; document.getElementById("sessConfirmBack").classList.remove("show"); });
document.getElementById("sessConfirmYes").addEventListener("click",confirmSessionUsed);
```

- [ ] **Step 4: Browser-verify the full profile flow (Playwright)**

Seed a client with a 6-session package, `openClientProfile(id)`:
1. Click "Mark session used" -> confirm modal shows "from 6 to 5"; click "Not yet" -> still 6.
2. Click again -> "Yes, mark it used" -> shows "5 of 6 sessions left"; reload page, reopen -> still 5 (persisted).
3. Set `#clpAppt` to `2026-07-08`, dispatch `change` -> `clpApptShown` shows "Wednesday, July 8, 2026" and `nextAppt` saved.
4. `clientUpdateMailto(currentClient())` starts with `mailto:` and contains the package line and the appointment.
5. Decrement to 0 -> row shows "Complete" and the button is disabled.

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "Add client profile modal: mark-used confirm, next appointment, email"
```

---

### Task 6: Backup / Restore all app data

**Files:**
- Modify: `index.html` — JS after Task 5's block; wires the `#backupBtn`, `#restoreBtn`, `#restoreFile` added in Task 4.

**Interfaces:**
- Consumes: the four storage keys `STORE_KEY` (`zeugma_services`), `TIER_KEY` (`zeugma_tiers`), `INV_KEY` (`zeugma_invoice_seq`), `CLIENTS_KEY` (`zeugma_clients`).
- Produces: `backupData()`, `restoreData(file)`.

- [ ] **Step 1: Add backup/restore JS**

```js
const BACKUP_KEYS = [STORE_KEY, TIER_KEY, INV_KEY, CLIENTS_KEY];
function invBackupStamp(){ const d=new Date(); const m=String(d.getMonth()+1).padStart(2,"0"); const day=String(d.getDate()).padStart(2,"0"); return ""+d.getFullYear()+"-"+m+"-"+day; }
function backupData(){
  const data={}; BACKUP_KEYS.forEach(k=>{ const v=localStorage.getItem(k); if(v!=null) data[k]=v; });
  const blob=new Blob([JSON.stringify({ app:"zeugma-savings", data }, null, 2)], {type:"application/json"});
  const url=URL.createObjectURL(blob);
  const a=document.createElement("a"); a.href=url; a.download="zeugma-backup-"+invBackupStamp()+".json";
  document.body.appendChild(a); a.click(); a.remove(); setTimeout(()=>URL.revokeObjectURL(url), 1000);
}
function restoreData(file){
  const reader=new FileReader();
  reader.onload=()=>{
    let parsed; try{ parsed=JSON.parse(reader.result); }catch(e){ alert("That file could not be read as a backup."); return; }
    const data=parsed && parsed.data; if(!data || typeof data!=="object"){ alert("That file is not a Zeugma backup."); return; }
    if(!confirm("Restore will replace all current data on this iPad with the backup. Continue?")) return;
    BACKUP_KEYS.forEach(k=>{ if(data[k]!=null) localStorage.setItem(k, data[k]); });
    location.reload();
  };
  reader.readAsText(file);
}
document.getElementById("backupBtn").addEventListener("click", backupData);
document.getElementById("restoreBtn").addEventListener("click", ()=>document.getElementById("restoreFile").click());
document.getElementById("restoreFile").addEventListener("change", e=>{ if(e.target.files[0]) restoreData(e.target.files[0]); e.target.value=""; });
```

- [ ] **Step 2: Browser-verify**

1. With some services/clients present, click **Back up** -> a `zeugma-backup-YYYY-MM-DD.json` downloads; open it and confirm it contains the four keys under `data`.
2. In console, build a backup object, `localStorage.clear()`, then feed the object through `restoreData` (via a `File`/`Blob`) and confirm after reload that services, discounts, and clients return. (Simplest: verify `BACKUP_KEYS` round-trips by calling the serialize half and re-writing the keys, then `loadClients()`/`loadServices()` are non-empty.)
3. Confirm a malformed file shows the "not a Zeugma backup" alert and changes nothing.

- [ ] **Step 3: Commit**

```bash
git add index.html && git commit -m "Add Back up / Restore for all app data"
```

---

## Self-Review

**Spec coverage:**
- `zeugma_clients` model + helpers -> Task 1. ✓
- Profile created on invoice send, match by email, top-up by name -> Task 1 (`upsertClientFromInvoice`) + Task 2 (hook). ✓
- Per-treatment counts -> Task 5 render. ✓
- Mark-used confirm "Has this session been done?" with before/after -> Task 5. ✓
- Next appointment manual + display -> Task 5. ✓
- Warm one-tap mailto email, appt line only when set, no em dashes -> Task 3. ✓
- Clients screen + search + nav -> Task 4. ✓
- Backup/Restore all data -> Task 6 (buttons added Task 4). ✓
- Out of scope items (auto email, cloud, history, undo, edit contact) -> not implemented. ✓

**Placeholder scan:** No TBD/TODO; every code step is complete; copy uses no em dashes; the Node email test asserts `—` absent.

**Type/name consistency:** `upsertClientFromInvoice(client, items)`, `loadClients`/`saveClients`, `pkgLeft`, `clientActivePkgs`, `findClient`, `clientUpdateMailto`, `formatApptDate`, `openClientProfile`, `buildClientsList`, `BACKUP_KEYS` are defined once and used with the same names/shapes across tasks. Client record fields (`id,name,email,phone,nextAppt,packages[{name,total,used}]`) are consistent throughout. Modal reveal uses the existing `.show` class; the confirm modal sits at `z-index:600` above the profile modal (`z-index:500`).

**Known cross-task note:** Task 4 wires card clicks to `openClientProfile`, which Task 5 defines. Implement Tasks 4 and 5 in order; between them, a card click throws (expected, resolved by Task 5).
