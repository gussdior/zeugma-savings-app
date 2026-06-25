# Client Profiles + Sessions-Left Email — Design Spec

**Date:** 2026-06-25
**Status:** Design approved via live mockups; spec under review

## Context

Zeugma sells session packages and bundles. After a sale they want to track how many
sessions each client has left and send the client a warm email update with their remaining
sessions and their next appointment date. The owner confirmed: profiles are created when an
invoice is sent, clients are found by name, the front desk marks sessions used manually, a
profile holds only the client's current packages/bundles, and the email is sent one-tap
from the staff's own iPad Mail (no server, consistent with the rest of the app).

The app is a single offline `index.html` with all state in `localStorage` (`zeugma_services`,
`zeugma_tiers`, `zeugma_invoice_seq`) and deployed via GitHub Pages. No backend.

## Decisions (confirmed with owner)

- **Email delivery:** one-tap from staff Mail via a `mailto:` link (plain text, no
  attachment), pre-filled with recipient, subject, and body. Staff taps Send.
- **Tracking granularity:** per treatment (e.g. "Laser Hair Removal: 3 of 6 left",
  "Hydrafacial: 2 of 4 left"). The email lists each treatment.
- **Mark session used:** guarded by a confirm popup "Has this session been done?" that shows
  the before to after count; only confirming decrements. No separate undo (the confirm is
  the safeguard).
- **Profile scope:** current packages only (no full purchase history).
- **Find client:** search by name on a new Clients screen.
- **Next appointment:** entered manually by staff on the profile; appears in the email only
  when set.
- **Storage + safety:** profiles in `localStorage`; add a **Backup / Restore** that exports
  all app data to a JSON file and restores from it (free, no server). Owner accepts that
  data lives on the one iPad and chose the backup button as the safety net.
- **Email tone:** warm, no em dashes (brand rule). Approved copy in the mockup.

## Data model

New `localStorage` key `zeugma_clients` holding an array of client objects:

```
{
  id:        string,        // stable id (derived from email)
  name:      string,
  email:     string,        // match key (lowercased, trimmed)
  phone:     string,
  nextAppt:  string|null,   // ISO date "2026-07-08" or null
  packages:  [ { name: string, total: number, used: number } ]
}
```

`sessionsLeft = total - used` per package. Helpers: `loadClients()` / `saveClients()`
mirroring the existing `loadServices()` / `saveServices()` pattern (try/catch around
`JSON.parse` / `setItem`).

## Components (all in `index.html`)

### 1. Profile creation on invoice send
After a successful invoice in `makeInvoicePDF()` (`index.html` invoice section), call
`upsertClientFromInvoice(client, items)`:
- Match an existing client by `email` (lowercased). If none, create one.
- For each invoice line item, upsert a package **by name**: if an active package with the
  same name exists, add the new sessions to its `total`; else append
  `{ name, total: qty, used: 0 }`.
- Save. This works for single-service and bundle invoices (each item is one treatment).

### 2. Clients screen (`#clients`)
A new full screen reached from a **Clients** button in the dashboard header (alongside
"Discounts" / "Add service"). Contains:
- A search box that filters the list by name (case-insensitive substring).
- A list of client cards (name, email, count of active packages, total sessions left).
- Tapping a card opens the **profile modal**.
- Header also holds **Back up** and **Restore** buttons (see §5).

### 3. Profile modal (`#clientBack`, reusing the `.modal-back`/`.modal` pattern)
Shows for the selected client:
- Name + email + phone.
- One row per package: name, "X of Y sessions left", a progress bar, and a
  **"− Mark session used"** button (disabled when `used >= total`, shown as "Complete").
- **Next appointment**: a native `<input type="date">`; storing updates `nextAppt`. Display
  the chosen date formatted ("Saturday, July 8, 2026").
- **"Email sessions update"** button (see §4).

### 4. Confirm-to-decrement + email
- **Mark session used** opens a confirm dialog (reuse modal pattern): title "Has this
  session been done?", body "Mark one <treatment> session as used for <name>. Sessions left
  will go from X to X-1.", buttons "Not yet" / "Yes, mark it used". Only confirm does
  `used++`, save, re-render the profile.
- **Email sessions update** builds a `mailto:` link:
  - `to` = client email; `subject` = "A little update on your Zeugma treatments".
  - Body (warm, approved tone, no em dashes): greeting, a line per active package
    ("<name>: X of Y sessions left"), the next-appointment sentence only if `nextAppt` set,
    a friendly close, and the Zeugma phone + site.
  - Open via `location.href = mailto:...` (encodeURIComponent on subject and body). On iPad
    this opens Mail pre-filled; staff taps Send.

### 5. Backup / Restore
- **Back up:** serialize all app `localStorage` keys (`zeugma_services`, `zeugma_tiers`,
  `zeugma_invoice_seq`, `zeugma_clients`) into one JSON object and download it as
  `zeugma-backup-YYYY-MM-DD.json` (Blob + temporary `<a download>`). On iPad this goes to
  Files / share sheet.
- **Restore:** a hidden `<input type="file">`; on select, parse JSON, write the keys back to
  `localStorage`, then reload the app. Guard with a confirm ("Restore will replace current
  data. Continue?").

## Out of scope (v1)

- Automatic/scheduled emails (would need a server; explicitly rejected).
- Cloud sync / multi-device sharing (same reason).
- Purchase history beyond current packages.
- Undo after confirming a session (the confirm dialog is the safeguard).
- Editing a client's name/contact after creation (created from the invoice; can revisit).

## Verification (Playwright, landscape)

1. Send a single-service invoice for a new email -> a client appears on the Clients screen
   with one package at full count (e.g. 0 used of 6).
2. Send a bundle invoice -> the client's profile shows one package per treatment with the
   adjusted session counts from the bundle.
3. Send another invoice for an existing email + same treatment -> that package's total tops
   up rather than duplicating.
4. "Mark session used" -> confirm popup shows correct before/after; "Not yet" leaves count
   unchanged; "Yes" decrements and persists across reload.
5. Enter a next appointment date -> profile shows it formatted; clearing it removes it.
6. "Email sessions update" -> the generated `mailto:` contains the right address, subject,
   each active package's "X of Y left", and the appointment line only when a date is set.
7. Search filters the client list by name.
8. Back up downloads a JSON containing all keys; Restore on a cleared store rebuilds
   services, discounts, and clients, then the app reflects them.
9. No console errors; existing dashboard, single-service, bundle, and invoice flows still
   work.
