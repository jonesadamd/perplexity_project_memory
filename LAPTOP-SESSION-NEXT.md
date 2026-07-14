# Laptop Session — Next Up (owner priorities, handed off from desktop 2026-07-14 evening)

> For the **laptop** machine. Read `SERENOVA-SESSION-HANDOFF.md` (🔝 TOP BRIEF, 2026-07-14 evening)
> + `SERENOVA-MACHINE-SYNC.md` first, then this. Four items below, in the owner's own priority order.
> Created by the desktop session at `0.7.0629` (code HEAD `18ffc17`), LIVE and verified.

## Standing rules (don't skip)
- `git pull --rebase` BOTH repos before starting AND before each push.
- Build: `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`. Bump `src/version.js`
  (`HUB_VERSIONS.serenova`) each deploy — next number is `0.7.0630`.
- Commit author **Adam Jones <aj@adamdcjones.com>** ONLY (NO Claude co-author trailer). Confirm the
  file list before any push.
- Deploy: `mv base44/entities base44/.entities-hidden && npx -y base44@0.0.56 --app-id
  68c22e8ff3726c063c4a53e2 site deploy -y && mv base44/.entities-hidden base44/entities`. Verify live
  via `curl` against the deployed `index-*.js` bundle, grep the version string.
- Local preview: `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite --port 5173 --host 0.0.0.0`
  (real auth + live data via the `/api` proxy).
- Base44 function count: 174 existing + up to 50 new = the cap. Repurpose an existing underused
  function (established convention — `diagnoseManagementAccess`/`syncTeamPaymentsForEvent` were both
  repurposed this way) rather than creating a new one where avoidable.

---

## ITEM 1 — Guest List Export dropdown redesign

**Goal (owner):** change the current "Export All" (one full-row button in a dropdown, plus one row
per show) into `Export All   [PDF] [CSV]` — the label on the left, separate small icon buttons for
PDF and CSV on the right, independently clickable. Same treatment for each per-show row.

**File:** `src/components/events/edit/GuestListTab.jsx` — the `exportMenu` JSX + `downloadCsv`
function (currently CSV-only, client-side Blob download). This item is really the UI shell; the CSV
half already works, the PDF half doesn't exist yet — see Item 2 for that.

---

## ITEM 2 — Clean, titled PDF export (By Show / By Date)

**Goal (owner):** an actual designed PDF — titled, clean layout — with two groupings: **By Show**
(today's existing per-performance grouping, unchanged) and **By Date** (owner-confirmed meaning: a
same-day rollup merging multiple shows on one calendar date, for days with 2+ sets).

**No PDF generation exists for guest lists today** (CSV only). Reuse the proven server-side jsPDF
pattern already live elsewhere — don't build a client-side jsPDF path:
- `base44/functions/generateFinancialSummaryPdf/entry.ts` — `new jsPDF({ format: 'letter' })`,
  clean single-purpose PDF generator, good template to copy the shape of.
- `base44/functions/generateFinancialPackage/entry.ts` — multi-document generator if you want a
  richer reference.

**Data shape:** `Event.guest_lists[]` — each entry has `performance_date`, `performance_time`,
`total_comps`, `guests[]` (`name`, `status`, `notes`, `requested_by_email`/`requested_by_name`,
`added_by_email`/`added_by_name` — the last pair is brand new this session, see Item 4's sibling
work in the handoff).

**Function budget:** mind the 50-new-function cap. Either add a new focused function
(`generateGuestListPdf`) if there's headroom, or repurpose an existing low-traffic one the same way
`diagnoseManagementAccess` was repurposed for Expenses.

---

## ITEM 3 — Public shareable Guest List link (+ retrofit expiry onto existing share links)

**Goal (owner):** a link you can send to anyone (e.g. a venue) to view the guest list — no login.
Clean, legible, presentable. Built-in PDF/CSV download, All or by Date.

**Model on the existing itinerary-share pattern (don't reinvent):**
- `base44/entities/ShareableLink.jsonc` — `link_type` enum needs a new value, e.g.
  `event_guest_list`.
- `base44/functions/generateShareableLink/entry.ts` — generic enough to reuse as-is (dedupes an
  existing active link per entity+type, else creates a random hex token).
- `base44/functions/validateShareableLink/entry.ts` — public, service-role token lookup, no auth.
- `base44/functions/getSharedItineraryData` — either extend with a new branch for the guest-list
  type, or copy its pattern into a new function (mind the function cap either way).
- `src/pages/SharedEventView.jsx` — the public route pattern (reads `?share_token=`); register a new
  route in `src/App.jsx` and extend the `isSharedItinerary`-style auth-bypass check (matches on
  pathname + presence of `share_token`) to also match the new route.
- `src/components/mobile/MobileShareDialog.jsx` — the share-dialog UX to mirror (calls
  `generateShareableLink`, builds `${origin}/SharedEventView?share_token=...&access_type=...`).

**Expiry rule (owner decision — doesn't exist for ANY share link today, build it):** final
performance/event date + 14 days. **Owner explicitly wants this retrofitted onto the EXISTING
itinerary share links too**, not just the new guest-list one. Compute it dynamically at validation
time (don't store a redundant `expires_at` field — derive from the linked event's/tour's own last
performance date, so it stays correct if dates ever change). Skip this rule entirely for
`link_type: 'staged_session'` links — those already have their own unrelated 30-day idle TTL
(`STAGED_IDLE_TTL_MS` pattern used in `syncTeamPaymentsForEvent`/`decideGuestRequest`/etc.) and
shouldn't get a second, conflicting expiry concept.

---

## ITEM 4 — SystemHub "Staged Users" browser UI

**Goal (owner):** click the Overview "Active Staged" KPI tile → see a list of every staged member
platform-wide, with active/inactive clearly visible, filterable.

**Backend is DONE, deployed this session** — `base44/functions/adminStagedSessions/entry.ts` gained
a `list_all` action: every staged identity platform-wide, grouped by email (a person can have
multiple device sessions), returns `{email, name, total_sessions, active_sessions, is_active,
last_active, devices}`, real names resolved via `User.list()` matched by email (same "fetch all,
match in JS" pattern `systemUsersSearch` already uses).

**No UI built yet.** Needs:
1. `src/components/systemhub/HubOverview.jsx` — the "Active Staged" KPI `Card` (in `KPI_CARDS`,
   currently zero `onClick` wiring at all on any KPI tile) needs an `onClick` that navigates to a new
   sub-tab. `onNavigate` prop is already just `setActiveTab` from `SystemHub.jsx` — you'll likely
   need a small combined handler (`setUsersSubTab('staged'); setActiveTab('users');`) passed down
   alongside it, mirroring `handleResultClick`'s existing pattern in `SystemHub.jsx` (~line 114-125).
2. `src/pages/SystemHub.jsx` — add a third `TabsTrigger`/`TabsContent` ("Staged Users") to the
   existing Users-tab `Tabs` (currently just "All Users"/"System Users", `usersSubTab` state already
   exists) — gate it the same way "System Users" is gated (`system_admin` only, since it's
   platform-wide/sensitive).
3. New component (e.g. `src/components/systemhub/StagedUsersManager.jsx`) — fetch `list_all` on
   mount, render a filter toggle (All/Active/Inactive), a table (name/email, active vs total
   sessions, last active, device labels), reuse the existing `revoke` action for a per-row
   "Force log out." Match the dark-slate styling already established in `AllUsersAdmin.jsx`
   (`bg-slate-800/60 border-slate-700`, `text-systemAdmin-400` accent, etc.).

**Separately flagged by owner, worth a product conversation not just code:** "All Users" requires
typing 2+ characters before showing anything; "System Users" shows its full list immediately, no
search needed. Researched: this is **not** a security boundary (`systemUsersSearch`'s backend fetches
ALL users/accounts into memory regardless of query length once past the gate; the 2-char minimum is
purely client-side, no stated rationale anywhere in the code) — it's an arbitrary UX choice. Owner
also wants staged users' active/inactive status visible directly in "All Users" search results, not
buried in the detail panel.

---

## Lower priority (logged, not scoped further)
The `NumberInput` (`src/components/financial/NumberInput.jsx`) snap-to-zero fix built this session
for the Financial Details page also needs wiring into `src/components/financial/TeamRateSettings.jsx`
and `src/components/financial/FinancialDefaults.jsx` (the Settings page) — same bug confirmed present,
just out of scope for the pass that fixed it elsewhere.

## After any item
Bump `src/version.js`, build green, confirm the file list with the owner before pushing, commit
(Adam Jones only, no Claude trailer), push, deploy via the Base44 CLI, verify live via curl, add a
decisions-log entry + bump build-phases, refresh this file + `SERENOVA-MACHINE-SYNC.md`'s pointers.
