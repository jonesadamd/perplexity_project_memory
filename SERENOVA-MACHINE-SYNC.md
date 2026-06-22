# Serenova Hub — Machine Sync & Coordination

> **Read this right after `SERENOVA-SESSION-HANDOFF.md`.** Two machines (desktop + laptop) each
> run an AI coding session against the same code repo (`jonesadamd/serenovahub_b44`) and share
> this memory repo (`jonesadamd/perplexity_project_memory`) via git. This doc is the lightweight
> protocol that keeps the two sessions from clobbering each other or duplicating work.
> Last Updated: 2026-06-22T02:15 EDT

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
| Code repo `main` HEAD | `d33ee3c` — "Venue Contacts polish: phone ext + country format, role-seed fix, expanded roles + Other" |
| Serenova version | **0.6.0568** |
| Memory repo HEAD | (this commit) |
| Build | green |
| Pending Base44 DEPLOYS (owner) | **(1)** redeploy fn **`getVenueRouteInfo`** · **(2)** re-sync **`Accommodation`** (`route_to_venue` + coord fields) · **(3)** sync **`Event`** (`primary_accommodation_id`) · **(4)** sync **`Venue`** (`contacts[].roles[]` + `contacts[].phone_ext`). Until done: #7 coordinate route + walk/drive cache, the primary-hotel dropdown persistence, and venue-contact role-tags/extension won't fully work live (everything else does). Geocoding API already enabled. |
| Pending live-verify | The whole **EventDetails redesign** (`0.6.0541`→`0.6.0568`) — owner reviewed most of it live during the build (route map, times, guest popup, contacts popup, venue dialog all confirmed "looks good"); just needs the Base44 deploys above to light up the gated bits. |

## 🟢 Active work (claim before building; clear when done)

| Machine | Task | Status | Notes |
|---|---|---|---|
| laptop | **Guest notifications + TM picker + mobile approve/deny** (Push Groups 1–2 + approver-notify + searchable multi-TM picker + linked-mgmt fix + mobile TM/Admin approve/deny) | **✅ DONE — owner-confirmed working 2026-06-21** | `0.6.0527`→`0.6.0533`. Full guest flow: request → notify TM(s)/AMC (email+push+`Notification`) → approve/deny on **web + mobile** → requester notified (email+push+banner). `TourManagerPicker` (searchable, multi-select, linked AMC/BMC people, artist-functional roles + User-entity names). `npm:web-push` proven on Deno. Mobile approve/deny (`0.6.0533`) verify on next deploy. **Push Group 3 (flight alerts) is separate — see below.** Docs: working doc + decisions-log v2.105, build-phases v2.63. |
| laptop | **Travel page — adding ground transportation issue** (FIX) | **✅ DONE — owner-confirmed working 2026-06-21** | TWO root causes fixed: (1) `0.6.0534` — link-only mgmt user 403'd `getUsersByEmailsForAccount` → Travel page fatal red-error; broadened that fn's access (also fixes TM-picker/Team enrichment for link-only mgmt) + made Travel enrichment non-fatal. (2) `0.6.0535` — Save no-op on ADD: `AddTravel` mis-wired `TravelForm` (React `ref` to a non-forwardRef fn that wants a `formRef` PROP; `?.save()` silent no-op; missing `accountId`/`onSave`) → fixed to `formRef`/`accountId`/`onSave`/`requestSubmit()` like EditTravel. Owner: "travel works and saves." |
| _(RELEASED by laptop 2026-06-22)_ | **EventDetails (web) redesign** — big stretch DONE; remaining sub-items unclaimed below | **🟢 RELEASED — major progress, owner on break** (`0.6.0541`→`0.6.0568`) | **Source of truth: `docs/EVENTDETAILS-REDESIGN-WORKING-DOC.md` + decisions log v2.134.** DONE: 4-tab scaffold + brand-teal Option-3 header; **Venue card** + **#7 route map** (walk/drive times, primary-hotel auto-pick + manual dropdown, coordinate route, key-contacts Advance/Tech); **Data Room** tab (Financials renamed + Files); **TM quick-access** in tab bar (multi + make-lead); **Management→People**; **itinerary full-width** + density/dedupe + ground-tz fix; **reference cards above itinerary, colorized**; **Guest List card→popup** redesign (open to all, color buttons, Export CSV); **All Contacts popup** (TM/Team/Venue/Mgmt, colorized); **#6 Full Venue Info dialog** rebuilt (tabbed, reads Venue entity) + **Venue Contacts** full redesign (show-all + link toggle + inline edit + multi-role tags + Other + phone ext + country format). **REMAINING (unclaimed):** #9+Stage 4 accommodation (Overview summary + hotel-grouped Travel list), #10 Set List popup, #7c all-hotels/airport. **+ the 4 Base44 deploys above.** |
| _(unclaimed)_ | EventDetails **#9 + Stage 4** — accommodation (Overview compact hotel summary + Travel hotel-grouped list, same-hotel-diff-dates consolidation) | next | spec in the redesign working doc (accommodation grouping section) |
| _(unclaimed)_ | EventDetails **#10** — Set List as a popup button (small; mirror the guest popup pattern) | backlog | — |
| _(unclaimed)_ | Push+Travel-alerts **Group 3** (GH-Actions cron + flight delay/gate alerts + on-entry lookup; builds `dispatchNotification`) | later | needs `NOTIFY_INTERNAL_SECRET` + a secret-gated `asServiceRole` entry; tiered cadence; idempotent alert dispatch |
| _(unclaimed)_ | B4.2 — Verify-gated phone/email change | backlog | re-key Memberships on email change; Verify code on phone change |
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
