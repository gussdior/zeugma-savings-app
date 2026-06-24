# Send Invoice — Design Spec

**Date:** 2026-06-24
**Status:** Approved invoice visual; design under review

## Context

After a client agrees to a package in the chair-side savings app, Zeugma wants to
hand them a proper invoice. Today the dashboard has one row per service with an
"Open for client" button; there is no way to produce a receipt. This feature adds a
**"Send invoice"** button next to "Open for client" on each service row (and on the
bundle screen), opens a short form to capture the client's details, and produces a
branded PDF invoice the user emails from the iPad in a couple of taps.

The invoice visual was designed and approved via live mockup (olive header with the
Montserrat "ZEUGMA MEDSPA" wordmark, website/Instagram, invoice no. + date, billed-to
block, a green always-on "PAID" stamp, an itemized package table, and a totals box
showing regular price / "you saved" / total, with a phone-number footer).

The app must stay a **single offline HTML file** with no build step and no new runtime
dependencies (consistent with how it ships today). Email needs internet, which is
expected and acceptable for the send step only.

## What we are building

1. A **"Send invoice" button** on each dashboard service row, beside `.open-btn`
   (`index.html:511`), and one on the bundle screen beside `#bCtaBtn` (`index.html:358`).
2. An **invoice form modal** (`#invoiceBack`) reusing the existing `.modal-back`/`.modal`
   pattern (`index.html:364`, `:386`) to collect client name, email, phone, and — for a
   single service — the number of sessions, with a live price preview.
3. An **invoice renderer** that fills a hidden `#invoiceSheet` element with the approved
   design, populated from real data.
4. **Print-to-PDF delivery:** a `@media print` stylesheet shows only `#invoiceSheet`; the
   user taps "Make PDF", iOS shows the print/share sheet, and they share to Mail (PDF
   auto-attached) or AirPrint. Fully offline-capable to generate; crisp vector text;
   zero libraries.

## Pricing — reuse, do not re-derive

The invoice MUST call the existing `calc(service, sessions)` (`index.html:630`) for every
number, so the invoice can never disagree with the presentation screen:

- Line item amount = `calc().packageTotal`
- Per-session = `calc().perSession`
- "Regular price" (struck) = `calc().regGross`
- "You saved" = `calc().totalSave`
- Total = `calc().packageTotal`

For a **bundle**, sum these across the selected services exactly as `bundleTotals()`
(`index.html:799`) already does — one itemized row per treatment.

## Components

### 1. Invoice number (date-based)
Format `ZMS-YYYYMMDD-N`, where N is the Nth invoice generated that day, starting at 1.
Stored in `localStorage` under a new key `zeugma_invoice_seq` as `{ date, seq }`. On
generate: if stored `date` equals today, `seq++`; otherwise reset `date=today, seq=1`.
The number is assigned when "Make PDF" is pressed and reused if the same invoice is
re-printed within that form session (so reprints don't burn a new number).

### 2. Invoice form modal `#invoiceBack`
Fields:
- **Client name** (required)
- **Client email** (required — needed to email it)
- **Client phone** (optional)
- **Sessions** (single-service only): number stepper, default `service.maxSessions`,
  clamped 3–12 via existing `clampSess` (`index.html:423`). Drives the live preview.
- **Live preview line:** "Package total $X · you save $Y" computed from `calc()`.

For a bundle, the form skips the sessions field; it uses each service's current
`maxSessions` (the same counts shown on the bundle screen). Basic validation: name and a
syntactically valid email required; show an error in the existing `.err` style.

### 3. Invoice renderer `buildInvoice(data)`
Writes the approved invoice markup into a hidden container `#invoiceSheet` (placed once at
the end of `<body>`). Inputs: business constants, invoice number, today's date (formatted
"Jun 24, 2026"), client info, and an array of line items (`{name, subtitle, qty,
perSession, amount}`) plus totals. Business constants (website, Instagram, phone
(305) 481-5659) live in one small config object near the top of the script so they are
easy to edit.

### 4. Delivery (`window.print()` + iOS share)
"Make PDF" calls `buildInvoice()` then `window.print()`. A `@media print` block hides
everything except `#invoiceSheet`. Resulting iPad flow:
**Send invoice → fill info → Make PDF → iOS share sheet → Mail (PDF attached) → confirm
the client's address → Send** (or AirPrint / Save to Files instead). The client's email
is shown on the confirmation step for easy entry. (A future upgrade could attach the PDF
to a pre-addressed email automatically, but that needs a bundled PDF library; out of
scope for v1.)

## Files touched
- `index.html` only — new modal markup, two buttons, `#invoiceSheet` container, invoice
  CSS (screen + `@media print`), and the invoice JS (number generator, form handlers,
  `buildInvoice`). No new files; nothing else changes.

## Out of scope (v1)
- Tax/VAT line (prices are all-in).
- Saving/printing a history of past invoices.
- True zero-tap automatic email (would require an email service + server).
- Embedding the logo as an image (the wordmark is rendered in Montserrat).

## Verification
1. Desktop browser: dashboard shows "Send invoice" on each row; clicking opens the form.
2. Fill name/email/sessions → preview total matches the client screen's total for the
   same service + session count (spot-check against `calc()` output).
3. "Make PDF" → browser print preview shows ONLY the branded invoice, correctly populated,
   with a `ZMS-YYYYMMDD-N` number and the PAID stamp.
4. Generate a second invoice the same day → number increments (…-2). Simulate a new day →
   resets to …-1.
5. Bundle: select 2+ services, open the bundle screen, "Send invoice" → invoice lists each
   treatment as its own row; totals equal `bundleTotals()`.
6. On an actual iPad: print preview → share → Mail shows the PDF attached. (Device-only
   check the user performs.)
