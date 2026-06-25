# Cloud Sync (Supabase) — Design Spec

**Date:** 2026-06-25
**Status:** Design approved; spec under review

## Context

Client profiles (sessions left, next appointment) currently live only in the iPad's
localStorage, with a manual Back up / Restore as the only safety net. The owner correctly
noted that a manual backup habit is unreliable. They chose **cloud sync**: data saved
automatically to a managed cloud database, safe from device loss and shared across iPads.

The app is a single offline `index.html` deployed on GitHub Pages (public), with all state
in localStorage. Cloud sync is its first outside connection, and it will hold client PII
(names, emails, phones), so security is the central concern.

## Decisions (confirmed with owner)

- **Provider:** Supabase (free tier covers one medspa's volume; managed, no server to run).
- **Login:** one **shared spa login** (email + password). Entered once per device; the
  session token is kept on that device. The password is NOT embedded in the public app.
- **What syncs:** all app data (clients, services/prices, discounts, invoice counter),
  stored as a **single JSON document** in one row. Simplest model, nothing gets lost,
  syncs across iPads.
- **Local-first / offline:** localStorage stays the working store. The app reads from it
  instantly and keeps working with no internet; changes push to the cloud when online and
  pull on open. The front desk is never blocked.
- **No new dependencies:** talk to Supabase via plain `fetch()` against its REST and Auth
  HTTP endpoints (no bundled library, preserving the single-file app).
- **Back up / Restore stays** as an extra manual safety net.

## Architecture

### Supabase project (owner sets up once, guided)
- One table `spa_state`:
  - `id text primary key` (always the literal `'main'` — single shared record)
  - `data jsonb not null` (the app-state bundle)
  - `updated_at timestamptz default now()`
- **Row Level Security ENABLED**, with policies that allow **only authenticated users** to
  `select` / `insert` / `update` the row, and **deny anonymous** entirely. (Exact SQL is
  produced and verified against the Supabase skill during implementation; the spec's
  requirement is: anon cannot read or write; authenticated can.)
- **Auth:** email+password enabled; one user created = the shared spa login.
- The project **URL** and **anon (public) key** are baked into `index.html`. These are safe
  to expose precisely because RLS denies all access without a valid login session.

### Client-side (all in `index.html`, plain fetch)
A small `cloud` layer with a clear interface:
- `cloudLogin(email, password)` -> POSTs `…/auth/v1/token?grant_type=password`; on success
  stores `access_token` + `refresh_token` + expiry in localStorage; returns ok/error.
- `cloudLogout()` -> clears tokens (local data remains).
- `cloudSession()` -> current tokens or null; `cloudRefresh()` -> refreshes an expired
  access token via `grant_type=refresh_token`.
- `cloudPull()` -> GET `…/rest/v1/spa_state?id=eq.main`; returns the stored `data` bundle
  (or null if none yet).
- `cloudPush(bundle)` -> upsert (POST with `Prefer: resolution=merge-duplicates`) the
  bundle to the `main` row.
- All requests send `apikey: <anon>` and `Authorization: Bearer <access_token>`; a 401
  triggers one `cloudRefresh()` + retry, then a re-login prompt if still failing.

### State bundle
`appBundle()` collects the existing keys into one object:
`{ zeugma_services, zeugma_tiers, zeugma_invoice_seq, zeugma_clients }` (reusing the
`BACKUP_KEYS` already defined for Back up / Restore). `applyBundle(obj)` writes them back to
localStorage. These already exist conceptually in the backup feature and are reused.

### Sync orchestration (local-first)
- **On app open:** if logged in and online, `cloudPull()`; if the cloud row is newer than
  local (compare a stored `syncedAt`), `applyBundle()` and re-render. If offline, just use
  local.
- **On any change** (the existing `saveServices`, `saveTiers`, `saveClients`, invoice
  counter writes): mark a `dirty` flag and schedule a **debounced** `cloudPush(appBundle())`
  (e.g. 1.5s after the last change). On success, clear `dirty` and record `syncedAt`.
- **Offline / failure:** keep `dirty=true`; retry on reconnect (`online` event) and when the
  app regains focus (`visibilitychange`). Nothing is lost locally meanwhile.
- **Conflict policy:** last-write-wins on the whole bundle. Acceptable for a single front
  desk; two iPads editing the same second is rare and the loser's change would be re-pushed
  on its next edit. Documented as a known limitation.

### UI
- **Login screen** (`#login`): shown on launch when there is no session; shared-login email
  + password, a Sign in button, and an error line. On success, hide and continue to the
  dashboard.
- **Sync status chip** in the dashboard header: "Synced" / "Saving…" / "Offline, will sync"
  / "Sign in" — so staff can see at a glance that data is safe.
- **Sign out** option (in the Clients screen header or a small menu).
- Login/sync copy follows the brand rule: no em dashes.

## Owner one-time setup (guided, click-by-click in the plan)
1. Create a free Supabase account and a new project.
2. In the SQL editor, paste one provided snippet (creates `spa_state`, enables RLS, adds
   the policies).
3. In Authentication, add one user = the shared spa login (email + password).
4. Copy the project **URL** and **anon public key** from Settings -> API and send them.
5. (Done by us) bake those two values into `index.html`, ship.

## Security notes
- Anon key public + RLS-deny-anonymous is Supabase's standard safe pattern; the key is a
  public identifier, not a secret.
- The shared password is the real gate: it is entered per device, stored only as a session
  token locally, and must be strong and shared only with front-desk staff.
- HTTPS throughout (GitHub Pages and Supabase are both HTTPS).

## Out of scope (v1)
- Per-staff accounts and audit of who did what.
- Live realtime updates between two simultaneously-open iPads (we pull on open + push on
  change; not websockets).
- Field-level/merge conflict resolution (whole-bundle last-write-wins).
- Syncing across different businesses / multi-tenant (single shared record).

## Verification
- **Sync orchestration** (debounce, dirty flag, pull-on-open newer-wins, offline retry):
  unit-style checks in Node/browser with `fetch` stubbed.
- **Auth + REST** against the **real project** once the owner completes setup: log in, send
  an invoice on iPad A, confirm it appears on iPad B after open; go offline, mark a session,
  go online, confirm it syncs; confirm a logged-out / anonymous request is rejected by RLS.
- Existing offline flows (dashboard, single, bundle, invoice, profiles, Back up/Restore)
  still work with no network.
- No console errors; no em dashes in new copy.
