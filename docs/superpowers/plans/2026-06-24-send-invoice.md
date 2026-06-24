# Send Invoice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-service and bundle "Send invoice" buttons that open a client-info form and produce a branded, print-to-PDF invoice the user emails from the iPad.

**Architecture:** All work is inside the single file `index.html`. Pure helpers (invoice number, line-item builders) reuse the existing `calc()` so invoice figures always match the presentation. A hidden `#invoiceSheet` is filled by `buildInvoice(data)` and revealed only by an `@media print` block; the iPad's print/share sheet handles emailing. A reused `.modal-back` form collects client details.

**Tech Stack:** Vanilla HTML/CSS/JS (no framework, no build, no dependencies), `localStorage`, browser print.

## Global Constraints

- Single offline file: **all changes go in `index.html`**. No new files, no libraries, no CDN additions, no build step.
- **No em dashes** in any shipped/user-visible copy (brand rule). Use commas, periods, or "to".
- Font is Montserrat everywhere; colors must use the existing palette vars (`--olive #4A3C0E`, cream `#E9E0CE`/`#efe7d2`, `--gold #C8A24C`, `--coral #C0664A`, sage-ink `#51662f`).
- **All invoice money figures must come from the existing `calc(service, sessions)` (`index.html:630`)** — never recompute prices independently.
- Business constants are fixed: website **zeugmaspa.com**, Instagram **@zeugmamedspa**, phone **(305) 481-5659**.
- Invoice number format: **`ZMS-YYYYMMDD-N`**, N starts at 1 each day.
- No test framework exists (matches the repo). Verify pure functions by pasting console snippets; verify UI/print by manual inspection. Commit after each task.
- Currency uses the existing `usd()` helper (`index.html:420`); escape user text with the existing `esc()` (`index.html:536`).

---

### Task 1: Business config, invoice-number + date helpers

**Files:**
- Modify: `index.html` — add constants/helpers in the script near the other constants (after `const CHECK_SVG`/`PEOPLE_SVG`, around `index.html:432`).

**Interfaces:**
- Produces:
  - `const BIZ = { web:"zeugmaspa.com", ig:"@zeugmamedspa", phone:"(305) 481-5659" }`
  - `nextInvoiceNo()` → string `"ZMS-YYYYMMDD-N"` (also persists the counter)
  - `peekInvoiceNo()` → string (computes the next number WITHOUT incrementing, for reprint reuse)
  - `invoiceDateStr(d?)` → string like `"Jun 24, 2026"`

- [ ] **Step 1: Add the config and helpers**

Insert after the `PEOPLE_SVG` constant (around `index.html:432`):

```js
const BIZ = { web:"zeugmaspa.com", ig:"@zeugmamedspa", phone:"(305) 481-5659" };
const INV_KEY = "zeugma_invoice_seq";
const INV_MONTHS = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];

function invoiceDateStr(d){
  d = d || new Date();
  return INV_MONTHS[d.getMonth()] + " " + d.getDate() + ", " + d.getFullYear();
}
function invStamp(d){
  d = d || new Date();
  const m = String(d.getMonth()+1).padStart(2,"0");
  const day = String(d.getDate()).padStart(2,"0");
  return "" + d.getFullYear() + m + day;
}
function invState(){
  try{ const s = JSON.parse(localStorage.getItem(INV_KEY)||"null"); if(s && s.date && s.seq) return s; }catch(e){}
  return { date:"", seq:0 };
}
function peekInvoiceNo(){
  const today = invStamp(), s = invState();
  const seq = (s.date === today) ? s.seq + 1 : 1;
  return "ZMS-" + today + "-" + seq;
}
function nextInvoiceNo(){
  const today = invStamp(), s = invState();
  const seq = (s.date === today) ? s.seq + 1 : 1;
  try{ localStorage.setItem(INV_KEY, JSON.stringify({ date:today, seq })); }catch(e){}
  return "ZMS-" + today + "-" + seq;
}
```

- [ ] **Step 2: Console-verify numbering and date**

Open `index.html` in a browser, open DevTools console, run:

```js
localStorage.removeItem("zeugma_invoice_seq");
console.assert(peekInvoiceNo().endsWith("-1"), "peek should be -1 on a fresh day");
const a = nextInvoiceNo(), b = nextInvoiceNo();
console.assert(a.endsWith("-1") && b.endsWith("-2"), "sequence should increment: " + a + "," + b);
console.assert(/^ZMS-\d{8}-\d+$/.test(a), "format ZMS-YYYYMMDD-N: " + a);
console.assert(invoiceDateStr(new Date(2026,5,24)) === "Jun 24, 2026", invoiceDateStr(new Date(2026,5,24)));
console.log("Task 1 OK", a, b);
```

Expected: no assertion errors; logs `Task 1 OK ZMS-...-1 ZMS-...-2`.

- [ ] **Step 3: Reset the counter so testing does not leak real numbers**

```js
localStorage.removeItem("zeugma_invoice_seq");
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add invoice config, date-based number generator, date helper"
```

---

### Task 2: Line-item builders (reuse calc)

**Files:**
- Modify: `index.html` — add after the `calc()` function (around `index.html:641`).

**Interfaces:**
- Consumes: `calc(s,q)` (`index.html:630`), `services`, `bundle`, `clampSess` (`index.html:423`).
- Produces:
  - `invoiceForService(s, q)` → `{ items:[{name,subtitle,qty,perSession,amount}], totals:{regGross,totalSave,packageTotal} }`
  - `invoiceForBundle()` → same shape, one item per bundled service

- [ ] **Step 1: Add the builders**

Insert immediately after `calc()` closes (around `index.html:641`):

```js
function invoiceForService(s, q){
  q = clampSess(q);
  const c = calc(s, q);
  return {
    items: [{
      name: s.name,
      subtitle: q + (q===1?" session":" sessions") + " package" + (c.hasPromo?" · promo price":""),
      qty: q,
      perSession: c.perSession,
      amount: c.packageTotal
    }],
    totals: { regGross: c.regGross, totalSave: c.totalSave, packageTotal: c.packageTotal }
  };
}
function invoiceForBundle(){
  const items = [];
  let regGross = 0, totalSave = 0, packageTotal = 0;
  bundle.forEach(k=>{
    const s = services.find(x=>x.key===k); if(!s) return;
    const q = clampSess(s.maxSessions), c = calc(s, q);
    items.push({
      name: s.name,
      subtitle: q + (q===1?" session":" sessions") + (c.hasPromo?" · promo price":""),
      qty: q, perSession: c.perSession, amount: c.packageTotal
    });
    regGross += c.regGross; totalSave += c.totalSave; packageTotal += c.packageTotal;
  });
  return { items, totals:{ regGross, totalSave, packageTotal } };
}
```

- [ ] **Step 2: Console-verify totals match calc()**

In the console:

```js
const s = services[0];
const r = invoiceForService(s, 6), c = calc(s, 6);
console.assert(r.totals.packageTotal === c.packageTotal, "service total mismatch");
console.assert(r.items[0].qty === 6 && r.items[0].amount === c.packageTotal, "service item mismatch");

bundle = [services[0].key, services[1].key];
const b = invoiceForBundle();
const expSave = calc(services[0], services[0].maxSessions).totalSave + calc(services[1], services[1].maxSessions).totalSave;
console.assert(b.items.length === 2, "bundle should have 2 rows");
console.assert(b.totals.totalSave === expSave, "bundle save mismatch");
bundle = [];
console.log("Task 2 OK", r.totals, b.totals);
```

Expected: no assertion errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add invoice line-item builders reusing calc()"
```

---

### Task 3: Invoice sheet renderer + print CSS

**Files:**
- Modify: `index.html` — add `#invoiceSheet` container just before `</body>` (the body ends after the script around `index.html:843`; place the empty container right before `<script>` at `index.html:402`); add CSS before `</style>` (`index.html:261`); add `buildInvoice()` in the script.

**Interfaces:**
- Consumes: `BIZ`, `esc()`, `usd()`, `invoiceDateStr()`.
- Produces: `buildInvoice(d)` where `d = { invoiceNo, dateStr, client:{name,email,phone}, items:[...], totals:{regGross,totalSave,packageTotal} }`. Renders into `#invoiceSheet`.

- [ ] **Step 1: Add the empty container**

Insert right before the `<script>` tag at `index.html:402`:

```html
<!-- ============ INVOICE (print only) ============ -->
<div id="invoiceSheet" aria-hidden="true"></div>
```

- [ ] **Step 2: Add CSS (screen-hidden + print-only + invoice styling)**

Insert before `</style>` (`index.html:261`):

```css
/* ---------- INVOICE ---------- */
#invoiceSheet{display:none;}
.inv-card{width:100%; max-width:720px; margin:0 auto; background:#fff; color:#3A3320; font-family:var(--sans);}
.inv-head{background:linear-gradient(135deg,#6f6442 0%,#4A3C0E 70%); color:#E9E0CE; padding:30px 40px; display:flex; justify-content:space-between; align-items:center; -webkit-print-color-adjust:exact; print-color-adjust:exact;}
.inv-word{font-size:22px; font-weight:400; letter-spacing:7px;}
.inv-contact{text-align:right; font-size:11px; color:#d3c9ad; line-height:1.7; letter-spacing:.5px;}
.inv-meta{padding:24px 40px 6px; display:flex; justify-content:space-between; align-items:flex-end;}
.inv-label-c{font-size:11px; font-weight:600; letter-spacing:2px; text-transform:uppercase; color:var(--coral); margin-bottom:4px;}
.inv-no,.inv-date{font-size:13px; color:var(--muted);}
.inv-date span{color:#3A3320; font-weight:600;}
.inv-billto{position:relative; padding:14px 40px 0;}
.inv-label{font-size:10px; font-weight:600; letter-spacing:1px; text-transform:uppercase; color:var(--muted); margin-bottom:6px;}
.inv-cli-name{font-size:16px; font-weight:600; color:var(--brown);}
.inv-cli-meta{font-size:13px; color:var(--muted); margin-top:2px;}
.inv-stamp{position:absolute; top:6px; right:46px; border:3px solid #51662f; color:#51662f; font-size:28px; font-weight:800; letter-spacing:5px; padding:5px 18px 7px; border-radius:9px; transform:rotate(-11deg); opacity:.72; text-transform:uppercase; box-shadow:inset 0 0 0 2px rgba(81,102,47,.25); -webkit-print-color-adjust:exact; print-color-adjust:exact;}
.inv-table{padding:26px 40px 0;}
.inv-row{display:grid; grid-template-columns:1fr 70px 110px 110px; align-items:center;}
.inv-thead{font-size:10px; font-weight:600; letter-spacing:.8px; text-transform:uppercase; color:var(--muted); padding:0 0 9px; border-bottom:2px solid var(--olive);}
.inv-table .inv-row:not(.inv-thead){padding:14px 0; border-bottom:1px solid #e7ddc8;}
.inv-it-name{font-size:15px; font-weight:600; color:var(--brown);}
.inv-it-sub{font-size:12px; color:var(--muted); margin-top:2px;}
.inv-c{text-align:center; font-size:14px;}
.inv-r{text-align:right; font-size:14px;}
.inv-amt{font-size:15px; font-weight:600;}
.inv-totals{padding:18px 40px 0; display:flex; justify-content:flex-end;}
.inv-tot-box{width:300px;}
.inv-tl{display:flex; justify-content:space-between; font-size:13px; color:var(--muted); padding:6px 0;}
.inv-strike{text-decoration:line-through;}
.inv-save{color:#51662f;} .inv-save span{font-weight:700;}
.inv-tot-final{display:flex; justify-content:space-between; align-items:center; margin-top:8px; padding:14px 16px; background:var(--olive); color:#fff; border-radius:10px; -webkit-print-color-adjust:exact; print-color-adjust:exact;}
.inv-tot-final span:first-child{font-size:13px; font-weight:600; letter-spacing:1px; text-transform:uppercase; color:var(--gold);}
.inv-tot-final span:last-child{font-size:24px; font-weight:800;}
.inv-foot{margin:26px 40px 30px; padding-top:18px; border-top:1px solid #e7ddc8; text-align:center;}
.inv-thanks{font-size:13px; color:var(--brown); font-weight:600;}
.inv-foot-sub{font-size:11px; color:#a99f80; margin-top:5px;}

@media print{
  @page{ margin:14mm; }
  body{ overflow:visible !important; }
  body > *{ display:none !important; }
  #invoiceSheet{ display:block !important; }
  #rotate{ display:none !important; }
}
```

- [ ] **Step 3: Add `buildInvoice()` in the script**

Insert after `invoiceForBundle()` (from Task 2):

```js
function buildInvoice(d){
  const rows = d.items.map(it=>`
    <div class="inv-row">
      <div><div class="inv-it-name">${esc(it.name)}</div><div class="inv-it-sub">${esc(it.subtitle)}</div></div>
      <div class="inv-c">${it.qty}</div>
      <div class="inv-r">${usd(it.perSession)}</div>
      <div class="inv-r inv-amt">${usd(it.amount)}</div>
    </div>`).join("");
  document.getElementById("invoiceSheet").innerHTML = `
    <div class="inv-card">
      <div class="inv-head">
        <div class="inv-word">ZEUGMA&nbsp;MEDSPA</div>
        <div class="inv-contact">${BIZ.web}<br>${BIZ.ig}</div>
      </div>
      <div class="inv-meta">
        <div><div class="inv-label-c">Invoice</div><div class="inv-no">No. ${esc(d.invoiceNo)}</div></div>
        <div class="inv-date"><span>Date:</span> ${esc(d.dateStr)}</div>
      </div>
      <div class="inv-billto">
        <div class="inv-label">Billed to</div>
        <div class="inv-cli-name">${esc(d.client.name)}</div>
        <div class="inv-cli-meta">${esc(d.client.email)}${d.client.phone?(" · "+esc(d.client.phone)):""}</div>
        <div class="inv-stamp">Paid</div>
      </div>
      <div class="inv-table">
        <div class="inv-row inv-thead">
          <div>Package / what's included</div>
          <div class="inv-c">Qty</div>
          <div class="inv-r">Per session</div>
          <div class="inv-r">Amount</div>
        </div>
        ${rows}
      </div>
      <div class="inv-totals">
        <div class="inv-tot-box">
          <div class="inv-tl"><span>Regular price</span><span class="inv-strike">${usd(d.totals.regGross)}</span></div>
          <div class="inv-tl inv-save"><span>You saved</span><span>${usd(d.totals.totalSave)}</span></div>
          <div class="inv-tot-final"><span>Total</span><span>${usd(d.totals.packageTotal)}</span></div>
        </div>
      </div>
      <div class="inv-foot">
        <div class="inv-thanks">Thank you for choosing Zeugma MedSpa.</div>
        <div class="inv-foot-sub">Questions about your package? Call us anytime at ${BIZ.phone}.</div>
      </div>
    </div>`;
}
```

- [ ] **Step 4: Manual verify the rendered invoice and print view**

Reload `index.html`. In the console:

```js
buildInvoice({
  invoiceNo:"ZMS-20260624-1", dateStr:invoiceDateStr(new Date(2026,5,24)),
  client:{name:"Jane Doe", email:"jane@email.com", phone:"(555) 123-4567"},
  ...invoiceForService(services[0], 6)
});
document.getElementById("invoiceSheet").style.display="block"; // temporary, to eyeball on screen
```

Confirm visually it matches the approved mockup (olive header + wordmark, contact on right, invoice no/date, billed-to + PAID stamp, one line row, totals box, phone footer). Then:

```js
document.getElementById("invoiceSheet").style.display=""; // restore
window.print();
```

Expected in print preview: ONLY the invoice shows (no dashboard, no rotate prompt); olive header and totals box keep their colors. Cancel the print dialog.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add print-only invoice sheet, styling, and buildInvoice renderer"
```

---

### Task 4: Invoice form modal (markup + handlers + buttons)

**Files:**
- Modify: `index.html` — add modal markup after the discount-settings modal (`index.html:400`); add the two buttons in `buildDashboard()` template (`index.html:511`) and the bundle CTA (`index.html:358`); add handlers + wiring in the script.

**Interfaces:**
- Consumes: `services`, `bundle`, `clampSess`, `usd`, `calc`, `invoiceForService`, `invoiceForBundle`, `buildInvoice`, `nextInvoiceNo`, `peekInvoiceNo`, `invoiceDateStr`.
- Produces: `openInvoiceForm(ctx)` where `ctx = {type:"service", key} | {type:"bundle"}`; `closeInvoiceForm()`; `makeInvoicePDF()`. Module-level `let invoiceCtx = null;`.

- [ ] **Step 1: Add the modal markup**

Insert after the discount-settings modal block closes (`index.html:400`, before `<script>`):

```html
<!-- ============ INVOICE FORM ============ -->
<div class="modal-back" id="invoiceBack">
  <div class="modal">
    <h2>Send invoice</h2>
    <div class="sub">Fill in the client's details, then make the PDF and email it from your iPad.</div>
    <div class="field"><label>Client name</label><input type="text" id="invName" placeholder="e.g. Jane Doe"></div>
    <div class="field-row">
      <div class="field"><label>Client email</label><input type="email" id="invEmail" inputmode="email" placeholder="jane@email.com"></div>
      <div class="field"><label>Client phone (optional)</label><input type="tel" id="invPhone" inputmode="tel" placeholder="(305) 555 0123"></div>
    </div>
    <div class="field" id="invSessWrap">
      <label>Sessions</label>
      <div class="stepper">
        <button class="step" id="invSessMinus" type="button">&#8722;</button>
        <span class="sess-val" id="invSessVal">6</span>
        <button class="step" id="invSessPlus" type="button">+</button>
      </div>
    </div>
    <div class="sub" id="invPreview" style="margin:4px 0 0; color:var(--brown); font-weight:600;"></div>
    <div class="err" id="invErr"></div>
    <div class="modal-actions">
      <button class="btn btn-ghost" id="invCancel" type="button">Cancel</button>
      <button class="btn btn-primary" id="invMake" type="button">Make PDF &amp; email</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Add the "Send invoice" button to each dashboard row**

In `buildDashboard()` replace the single open button (`index.html:511-513`) so the two buttons sit together:

```js
      <div class="row-cta">
        <button class="invoice-btn" data-invoice="${s.key}">Send invoice</button>
        <button class="open-btn" data-open="${s.key}">Open for client
          <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><polyline points="9 18 15 12 9 6"/></svg>
        </button>
      </div>
```

- [ ] **Step 3: Add CSS for the new button + wrapper**

Insert near the `.open-btn` styles (search `.open-btn{` in `<style>`) — add:

```css
.row-cta{display:flex; align-items:center; gap:10px; flex-wrap:wrap;}
.invoice-btn{font-family:inherit; font-size:14px; font-weight:600; padding:11px 18px; border:1px solid var(--line); border-radius:30px; background:#fff; color:var(--brown); cursor:pointer; white-space:nowrap; box-shadow:var(--shadow-sm); transition:background .2s, transform .12s;}
.invoice-btn:hover{background:var(--cream);}
.invoice-btn:active{transform:scale(.97);}
```

- [ ] **Step 4: Wire the row button**

In `buildDashboard()`, next to the existing `.open-btn` wiring (`index.html:530`), add:

```js
  wrap.querySelectorAll("[data-invoice]").forEach(b=> b.addEventListener("click",()=>openInvoiceForm({type:"service", key:b.dataset.invoice})) );
```

- [ ] **Step 5: Add the bundle-screen "Send invoice" button**

In the bundle CTA (`index.html:358-360`), change the `.cta-wrap` so it has both actions:

```html
    <div class="cta-wrap"><button class="cta-btn" id="bCtaBtn">Reserve this package
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><line x1="5" y1="12" x2="19" y2="12"/><polyline points="12 5 19 12 12 19"/></svg>
    </button><button class="invoice-btn" id="bInvoiceBtn" type="button" style="background:rgba(255,255,255,.1); color:#efe7d2; border-color:#6b5a22;">Send invoice</button><div class="note">Lock in your savings today.</div></div>
```

- [ ] **Step 6: Add the form logic (open/close/stepper/preview/make)**

Insert near the other modal handlers in the script (e.g. after the bundle handlers around `index.html:841`):

```js
let invoiceCtx = null;
const invEmailOk = e => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test((e||"").trim());

function invoiceData(){
  const built = invoiceCtx.type === "bundle"
    ? invoiceForBundle()
    : invoiceForService(services.find(s=>s.key===invoiceCtx.key), invoiceCtx.sessions);
  return {
    client:{ name:document.getElementById("invName").value.trim(),
             email:document.getElementById("invEmail").value.trim(),
             phone:document.getElementById("invPhone").value.trim() },
    items: built.items, totals: built.totals
  };
}
function refreshInvoicePreview(){
  const d = invoiceData();
  document.getElementById("invPreview").textContent =
    "Package total " + usd(d.totals.packageTotal) + " · you save " + usd(d.totals.totalSave);
}
function openInvoiceForm(ctx){
  invoiceCtx = ctx;
  if(ctx.type === "service"){
    const s = services.find(x=>x.key===ctx.key);
    invoiceCtx.sessions = clampSess(s.maxSessions);
    document.getElementById("invSessWrap").style.display = "";
    document.getElementById("invSessVal").textContent = invoiceCtx.sessions;
  } else {
    document.getElementById("invSessWrap").style.display = "none";
  }
  document.getElementById("invName").value = "";
  document.getElementById("invEmail").value = "";
  document.getElementById("invPhone").value = "";
  document.getElementById("invErr").textContent = "";
  invoiceCtx.invoiceNo = null;
  refreshInvoicePreview();
  document.getElementById("invoiceBack").classList.add("show");
}
function closeInvoiceForm(){
  document.getElementById("invoiceBack").classList.remove("show");
  invoiceCtx = null;
}
function invStep(d){
  if(!invoiceCtx || invoiceCtx.type !== "service") return;
  invoiceCtx.sessions = clampSess(invoiceCtx.sessions + d);
  document.getElementById("invSessVal").textContent = invoiceCtx.sessions;
  refreshInvoicePreview();
}
function makeInvoicePDF(){
  const name = document.getElementById("invName").value.trim();
  const email = document.getElementById("invEmail").value.trim();
  const err = document.getElementById("invErr");
  if(!name){ err.textContent = "Please enter the client's name."; return; }
  if(!invEmailOk(email)){ err.textContent = "Please enter a valid client email."; return; }
  err.textContent = "";
  if(!invoiceCtx.invoiceNo) invoiceCtx.invoiceNo = nextInvoiceNo();
  const d = invoiceData();
  buildInvoice({ invoiceNo:invoiceCtx.invoiceNo, dateStr:invoiceDateStr(), client:d.client, items:d.items, totals:d.totals });
  closeInvoiceForm();
  setTimeout(()=>window.print(), 60);
}

document.getElementById("invCancel").addEventListener("click", closeInvoiceForm);
document.getElementById("invoiceBack").addEventListener("click", e=>{ if(e.target.id==="invoiceBack") closeInvoiceForm(); });
document.getElementById("invMake").addEventListener("click", makeInvoicePDF);
document.getElementById("invSessMinus").addEventListener("click", ()=>invStep(-1));
document.getElementById("invSessPlus").addEventListener("click", ()=>invStep(1));
["invName","invEmail","invPhone"].forEach(id=> document.getElementById(id).addEventListener("input", ()=>{ if(document.getElementById("invErr").textContent) document.getElementById("invErr").textContent=""; }));
document.getElementById("bInvoiceBtn").addEventListener("click", ()=>openInvoiceForm({type:"bundle"}));
```

- [ ] **Step 7: Manual end-to-end verify**

Reload `index.html`. Then:
1. Dashboard: each service row shows **Send invoice** next to **Open for client**.
2. Click **Send invoice** on a service → modal opens; preview reads "Package total $X · you save $Y". Press +/- on Sessions → preview updates and matches the client screen for that service + count.
3. Leave name blank, click **Make PDF & email** → inline error "Please enter the client's name."; enter name, bad email → email error; fix both → print preview shows the populated invoice; cancel.
4. Generate one, then another the same day → invoice numbers end `-1` then `-2` (check the printed "No."). Reset after: `localStorage.removeItem("zeugma_invoice_seq")`.
5. Bundle: tick 2+ services, open the bundle screen, click **Send invoice** → modal has no Sessions field; print preview lists each treatment as its own row; totals equal the bundle screen's "You save".

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add Send invoice form, dashboard + bundle buttons, and PDF/print flow"
```

---

## Self-Review

**Spec coverage:**
- Per-service button → Task 4 Step 2/4. Bundle button → Task 4 Step 5/6. ✓
- Client-info form (name/email/phone/sessions + preview) → Task 4. ✓
- Reuse of `calc()` for all figures → Task 2 (builders) + Task 3 (render). ✓
- Date-based invoice number → Task 1. ✓
- Approved invoice visual (wordmark, contact, PAID stamp, totals, phone footer) → Task 3. ✓
- Print-to-PDF delivery via iOS share → Task 3 CSS + Task 4 `makeInvoicePDF`. ✓
- Real business constants (zeugmaspa.com / @zeugmamedspa / phone) → Task 1 `BIZ`. ✓
- Out of scope (tax, history, auto-email, image logo) → not implemented. ✓

**Placeholder scan:** No TBD/TODO; every code step shows full code; copy contains no em dashes. ✓

**Type/name consistency:** `invoiceForService`/`invoiceForBundle` return `{items, totals:{regGross,totalSave,packageTotal}}`, consumed unchanged by `buildInvoice` and `refreshInvoicePreview`. `invoiceCtx` shape `{type, key?, sessions?, invoiceNo?}` consistent across open/step/make. Helpers `nextInvoiceNo`/`peekInvoiceNo`/`invoiceDateStr`/`BIZ`/`esc`/`usd`/`clampSess` all defined before use. ✓

**Note for implementer:** Verified — modals in this file reveal via the `show` class (`.modal-back.show{display:flex}` at `index.html:102`; `openModal`/`closeModal` use `classList.add/remove("show")` at `index.html:569,572`). `#invoiceBack` follows the same pattern, so no special handling is needed.
