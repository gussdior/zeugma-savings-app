# Add Client + Optional Login — Design Spec

**Date:** 2026-06-25
**Status:** Self-approved (owner delegated: "finish everything, don't wait for my approval", leaving to use the app at the medspa)

## Context

Two needs, decided together because the owner is about to depend on the live app:

1. **Add client manually.** Profiles are only created when an invoice is sent, so existing
   clients (who bought packages before the app) cannot be entered. Owner wants an "Add
   client" form, including their already-used sessions.
2. **No lockout risk.** Cloud sync just went live with a hard login wall, but the owner has
   not verified the front-desk password works and is heading to a live shift. A broken
   login would lock them out of their own tool. We remove that risk by making sign-in
   optional rather than a hard gate.

## Decisions

### Add client
- **Entry points:** an "Add client" button in the Clients screen header; an "Add your first
  client" prompt in the empty state.
- **Form** (`#addClientBack`, mirrors the Add service modal): Client name (required), Email
  (required, valid), Phone (optional), then one or more **treatment rows** — Treatment name,
  Total sessions, **Sessions used** (owner's chosen input). A "+ Add another treatment"
  button adds rows (bundles). Treatment name is free text with the existing service names as
  `<datalist>` suggestions.
- **Save / validation:** name present, email valid, each treatment has a name + total >= 1,
  used clamped to 0..total. Then **upsert by email**: if the email exists, append the
  treatments to that client (also the way to add a package to someone later); else create a
  new client. Persists via `saveClients` (which already triggers cloud sync).
- New helper `addClientManual({name,email,phone}, packages)` where `packages` is
  `[{name,total,used}]`.

### Optional login (lockout fix)
- Startup still shows the login screen first when cloud is enabled and there is no session,
  but the login screen gains a **"Continue without signing in"** action that drops into the
  app local-only (no sync).
- When cloud is enabled and not signed in, the dashboard shows the sync chip as a clickable
  **"Sign in"** that reopens the login screen, so sync can be turned on anytime.
- Security unchanged: the cloud data is still RLS-protected (no session = no cloud read).
  Working without signing in only uses this device's local data, exactly as the app behaved
  before cloud sync. A fresh device with no local data and no login shows an empty app, not
  someone else's clients.

## Out of scope
- Editing an existing client's name/phone inline (re-adding by same email can append
  packages; full edit can come later).
- Removing or editing individual packages from a profile.

## Verification (Playwright + Node)
1. Add a brand-new client with two treatments (one with used > 0) -> appears in the list;
   profile shows correct "X of Y left" per treatment.
2. Add again with the same email -> treatments append to the existing client, no duplicate.
3. Validation: blank name, bad email, total < 1, used > total each blocked with a message.
4. Optional login: with cloud enabled + no session, login screen shows; "Continue without
   signing in" reaches the dashboard; the chip shows clickable "Sign in"; clicking returns
   to login. Signing in (stubbed) shows "Synced".
5. No console errors; existing flows (invoice, profiles, bundle, sync chip) still work.
