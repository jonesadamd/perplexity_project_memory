# Serenova Hub — Machine Sync & Coordination

> **Read this right after `SERENOVA-SESSION-HANDOFF.md`.** Two machines (desktop + laptop) each
> run an AI coding session against the same code repo (`jonesadamd/serenovahub_b44`) and share
> this memory repo (`jonesadamd/perplexity_project_memory`) via git. This doc is the lightweight
> protocol that keeps the two sessions from clobbering each other or duplicating work.
> Last Updated: 2026-06-22T22:35 EDT
> **👉 NEXT (owner priority):** see `LAPTOP-SESSION-NEXT.md` — (1) ~~MobileHub home install/notifications
> nudge~~ ✅ DONE (`0.6.0576`), (2) **live flight updates in MobileHub (Lisa flies 2026-06-23)** — still
> unclaimed below.
> **NEW CONVENTION (2026-06-22): authorship signature.** Repo now has `LICENSE` (proprietary) +
> `SIGNATURE.md` (overt RUSHMERE/KABINGA markers) + a CLAUDE.md "Authorship" pointer. Markers are
> display-only, overt, removable — and **NEVER in security- or randomness-sensitive code** (no
> seeds/salts/nonces/tokens/IDs). No markers applied to existing code yet (files + pointer only).

---

## 🔴 Golden rules (both machines, every time)

1. **Pull before you touch anything.** At the START of a work chunk, on BOTH repos:
   `git pull --rebase` (Base44 also 2-way-syncs the code repo, so this is doubly important).
2. **Pull again right before you push** (both repos). Resolve any trivial rebase conflict.
3. **Claim your task before building.** Update the **Active work** table below (set the machine +
   task + "in progress"), commit + push THIS file first. That tells the other machine what's taken.
4. **Push the handoff + this file at the end of every work chunk**, not just at session end — the
   other machine only sees what's been pushed.
5. **One track per machine at a time.** Don't both edit the same files/feature. If the next two
   tracks are independent (e.g. B4.2 vs the Push planning doc), split them; otherwise serialize.
6. **The on-screen version is the source of truth for "what's deployed."** Bump `src/version.js`
   every deploy; never assume the other machine's push is live until the canary shows on screen.
7. **Commits:** author **Adam Jones <aj@adamdcjones.com>** + the Claude co-author trailer.
   Confirm the file list before any push.

## ⚙️ Build / deploy (unchanged — repeated here so it's one read)
- Build: `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build` (plain `npx vite build`
  may grab the wrong vite).
- Base44 **"Synced" ≠ deployed** → force a rebuild via the editor **add-space-Save** trick, then
  hard-reload. New Base44 functions/entities must **provision** (watch the non-provisioning gotcha).
- Bump the affected hub in `src/version.js` each deploy; verify the canary on screen.

---

## 📍 Live pointers (update on every push)

| What | Value |
|---|---|
| Code repo `main` HEAD | `de6123c` — "MobileHub personal scope: strict tabs, reservation drawers, summary collapse" |
| Serenova version | **0.6.0578** |
| Memory repo HEAD | (this commit) — **NOTE:** local memory clone is now `/Users/adamjones/Developer/perplexity_project_memory` (the `/Volumes/adamjones/...` external mount is gone). |
| Build | green |
| Base44 DEPLOYS | ✅ **ALL 4 DONE + verified live 2026-06-22** — `getVenueRouteInfo` redeployed (coordinate route works); `Accommodation` (`route_to_venue`+coords), `Event` (`primary_accommodation_id`), `Venue` (`contacts[].roles[]`+`contacts[].phone_ext`) synced. Coordinate route, primary-hotel dropdown persistence, and venue-contact role-tags/extension all confirmed working. |
| Base44 DEPLOYS | ✅ **#7c Part B deployed + VERIFIED LIVE 2026-06-22** — owner confirmed "distance to airport is showing." `getVenueRouteInfo` re-provisioned + `Accommodation.route_to_airport` synced. (Prior 4 EventDetails deploys also done + verified.) |
| Pending live-verify | **#10 + follow-ups** (`0.6.0571`) — rebuild for the canary: Set List summary card → popup (Add/Edit/**Delete** for admin/TM/owner now that `role_templates.perm_setlist`='full' for artist super_admin/admin/tour_manager; band_member stays edit); new set lists default-name "{venue} - {Month Year}"; Contacts popup no longer scrolls sideways. **Permission change is a live Supabase row update (no deploy)** — takes effect on next permission refresh/re-login. |

## 🟢 Active work (claim before building; clear when done)

| Machine | Task | Status | Notes |
|---|---|---|---|
| laptop | **Guest notifications + TM picker + mobile approve/deny** (Push Groups 1–2 + approver-notify + searchable multi-TM picker + linked-mgmt fix + mobile TM/Admin approve/deny) | **✅ DONE — owner-confirmed working 2026-06-21** | `0.6.0527`→`0.6.0533`. Full guest flow: request → notify TM(s)/AMC (email+push+`Notification`) → approve/deny on **web + mobile** → requester notified (email+push+banner). `TourManagerPicker` (searchable, multi-select, linked AMC/BMC people, artist-functional roles + User-entity names). `npm:web-push` proven on Deno. Mobile approve/deny (`0.6.0533`) verify on next deploy. **Push Group 3 (flight alerts) is separate — see below.** Docs: working doc + decisions-log v2.105, build-phases v2.63. |
| laptop | **Travel page — adding ground transportation issue** (FIX) | **✅ DONE — owner-confirmed working 2026-06-21** | TWO root causes fixed: (1) `0.6.0534` — link-only mgmt user 403'd `getUsersByEmailsForAccount` → Travel page fatal red-error; broadened that fn's access (also fixes TM-picker/Team enrichment for link-only mgmt) + made Travel enrichment non-fatal. (2) `0.6.0535` — Save no-op on ADD: `AddTravel` mis-wired `TravelForm` (React `ref` to a non-forwardRef fn that wants a `formRef` PROP; `?.save()` silent no-op; missing `accountId`/`onSave`) → fixed to `formRef`/`accountId`/`onSave`/`requestSubmit()` like EditTravel. Owner: "travel works and saves." |
| _(RELEASED by laptop 2026-06-22)_ | **EventDetails (web) redesign** — big stretch DONE; remaining sub-items unclaimed below | **🟢 RELEASED — major progress, owner on break** (`0.6.0541`→`0.6.0568`) | **Source of truth: `docs/EVENTDETAILS-REDESIGN-WORKING-DOC.md` + decisions log v2.134.** DONE: 4-tab scaffold + brand-teal Option-3 header; **Venue card** + **#7 route map** (walk/drive times, primary-hotel auto-pick + manual dropdown, coordinate route, key-contacts Advance/Tech); **Data Room** tab (Financials renamed + Files); **TM quick-access** in tab bar (multi + make-lead); **Management→People**; **itinerary full-width** + density/dedupe + ground-tz fix; **reference cards above itinerary, colorized**; **Guest List card→popup** redesign (open to all, color buttons, Export CSV); **All Contacts popup** (TM/Team/Venue/Mgmt, colorized); **#6 Full Venue Info dialog** rebuilt (tabbed, reads Venue entity) + **Venue Contacts** full redesign (show-all + link toggle + inline edit + multi-role tags + Other + phone ext + country format). **REMAINING (unclaimed):** #9+Stage 4 accommodation (Overview summary + hotel-grouped Travel list), #10 Set List popup, #7c all-hotels/airport. **+ the 4 Base44 deploys above.** |
| laptop | EventDetails **#9 + Stage 4** — accommodation | **✅ DONE 2026-06-22** (`0.6.0569`) | Travel tab `AccommodationCard` now **grouped by hotel** (`src/utils/accommodationGroups.js`): one listing per hotel (name+city) + a row per stay (`check-in → check-out` date-only + occupants from `rooming_list[].occupants[]`); same hotel/diff dates = one listing, multiple rows. #9 Overview summary already covered by the venue card's primary-hotel line. Decisions log v2.135. |
| desktop | EventDetails **#10** — Set List as a popup button (mirror the guest popup) + tidy | **✅ DONE 2026-06-22** (`0.6.0570`, commit `f9871d5`) | `SetListCard` → summary button (sets/songs count, purple top-border) → new `SetListManagerDialog` (read/quick popup; privileged Add/per-set Edit open the existing `SetListDialog` as a sibling — no nested dialogs). Removed unused `getStatusBadge`/`getContractBadge` + `Badge` import from `EventDetails.jsx`. No backend changes → just a rebuild for the canary. |
| desktop | EventDetails **#7c** — >1-hotel driving-only + hotel→airport leg | **✅ DONE 2026-06-22** (`0.6.0572`→`0.6.0573`) | **Part A** (`0.6.0572`, `2d3b9bc`, frontend-only): venue card "+N more" → toggle → all-hotels list w/ drive-time-to-venue (cached `getVenueRouteInfo` per hotel). **Part B** (`0.6.0573`, `7f761ec`): `getVenueRouteInfo` now also computes **hotel→airport** (driving), cached as new additive `Accommodation.route_to_airport`; airport resolved server-side from the event's flights (departure_iata of latest-departing flight; fallback most-frequent). Shown in primary readout + all-hotels list (Plane icon). **✅ Part B deployed + VERIFIED LIVE 2026-06-22** (owner: "distance to airport is showing"). |
| _(unclaimed)_ | Push+Travel-alerts **Group 3** (GH-Actions cron + flight delay/gate alerts + on-entry lookup; builds `dispatchNotification`) | later | needs `NOTIFY_INTERNAL_SECRET` + a secret-gated `asServiceRole` entry; tiered cadence; idempotent alert dispatch |
| desktop | **MobileHub improvement #1** — venue route + drive/airport times on the mobile event detail | **✅ DONE 2026-06-22** (`0.6.0574`, `aa22b4e`) | Reuses the deployed `getVenueRouteInfo` (cached `route_to_venue`/`route_to_airport` on Accommodation). Event-detail Venue tab → "Getting There" block: drive/walk to venue + hotel→airport + tap-to-navigate. Works staged + logged-in (fn invoke); seeded from cached fields. Frontend-only. **Follow-up `0.6.0575` (`d537c5a`):** mobile Venue tab **contacts were empty** — read the legacy `venue_snapshot.day_of_contacts` but the redesign moved venue contacts to `venue.contacts[]`; now reads `venue.contacts` (scoped to `day_of_show_contact_ids`; snapshot fallback) + handles multi-role `roles[]`; renders below "Getting There". **From the MobileHub-improvements list (chat 2026-06-22); more there: TM quick-call, notifications inbox, offline day-sheet, Hotels grouping, etc.** |
| desktop | **MobileHub — band/crew see only their OWN travel/hotels** (Travel tab, Hotel tab, home day-schedule/week-strip); event detail pins+highlights own, collapses others | **✅ DONE 2026-06-22** (`0.6.0577`→**`0.6.0578`**, `0497a6a`→`de6123c`) | New `src/utils/personalTravelScope.js` (match: flights `user_email`, ground `travelers[].user_email`, hotels `rooming_list[].occupants[].email` + name fallback). Applies to `band_member`/`crew_member`/`roadie_member` only; mgmt/TM keep full view; events stay rostered. **Round 2 (`0.6.0578`, live-test refinements):** Travel+Hotel tabs now **STRICT** (reversed the per-event hotel fallback — owner wanted only-what-you're-assigned-to). **Detail drawers:** "My Reservation" (flights grouped by shared booking ref) + "Other Passengers"/"Others Also Here" name lists (`splitFlightBookings`/`splitRooms`; travel tab passes FULL bookings to drawer so co-passengers resolve). "You" tag → solid **teal-500**. Event detail: on-nothing-in-section → "{who} is flying · Show flights" summary (`CollapsedSummary`), else pin-own + "Show N other …". Frontend-only. ⏳ live-verify. |
| desktop | **MobileHub home nudge** — "Add to Home Screen" (if not a PWA) + "Enable notifications" (if PWA & not granted); dismissible, reappear ~every 3 days | **✅ DONE 2026-06-22** (`0.6.0576`, `7a93173`) | New `src/components/mobile/MobileHomeNudge.jsx` at top of Home tab (above guest notices/week strip): ONE banner — install (native prompt Android/Chrome; iOS instruction sheet) OR enable-notifications (`enablePush()`, reuses Push Group 1). Snooze ~3 days/type via localStorage (`nudge_install_until`/`nudge_push_until`); push only nudged while permission `'default'`. Frontend-only — plain rebuild, no fn/entity deploy. Live-only on app.serenovahub.com. ⏳ live-verify pending (canary `v0.6.0576`). |
| _(unclaimed — ⭐ OWNER PRIORITY for laptop, Lisa flies 2026-06-23)_ | **Live flight updates in MobileHub** — delay/gate/status on the mobile flight drawer (Lisa = logged-in, has the PWA) | next | **Full spec: `LAPTOP-SESSION-NEXT.md` Item 2.** Reuse web `LiveFlightTracker.jsx` + `getUserUpcomingFlightsWithLiveData`/`fetchLiveFlightData` (`AEROAPI_KEY_LIVE`, day-of window) → status block in `MobileTravelDetailDrawer`. On-demand-on-open for the demo; precursor to Push Group 3. Staged path = follow-up. |
| _(unclaimed — ON HOLD)_ | B4.2 — Verify-gated phone/email change | ⏸️ **paused by owner 2026-06-22** (re-key risk) | Plan is ready (in decisions log / chat): 2 fns `requestStagedContactChange`+`confirmStagedContactChange` (Twilio Verify to the NEW value) → phone updates `phone_mobile`; email **re-keys** `user_email` across all Memberships + re-points the staged-session `ShareableLink`. Owner held it pending more confidence in the email re-key. Pick back up when ready. |
| _(unclaimed)_ | Phase SA C part 2 — Expenses (`StagedRequests`) | backlog (owner: not yet) | holding queue + receipt quarantine |

> When you pick up a task: put your machine name in the row, set Status to "in progress",
> commit+push this file, THEN start. When done, set "done <date>" and update the pointers table.

---

## Repos & locations
- **Code:** `jonesadamd/serenovahub_b44` (`main`) — desktop clone at
  `/Users/adamjones/Developer/serenovahub_b44` (laptop path may differ).
- **Memory:** `jonesadamd/perplexity_project_memory` (`main`) — this folder. Holds the handoff +
  this sync doc. The repo `/docs` folder (in the code repo) is the source of truth for decisions/
  architecture/build status; this memory repo is the cross-session quick-load + coordination layer.
