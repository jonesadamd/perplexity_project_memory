# Serenova Hub — Session Handoff
> Quick-load context for any new AI session working on this project.
> **Always read this first. Then read the repo docs before touching any code.**
> **Two-machine setup (desktop + laptop) share this memory repo via git — read
> `SERENOVA-MACHINE-SYNC.md` (same folder) before starting so the two sessions don't collide.**
> Last Updated: 2026-07-17T09:20 EDT

---

## 🔝 TOP BRIEF — read FIRST (2026-07-17 · supersedes everything below)

> **LIVE = Serenova `0.7.0660` (deployed + verified). Repo `main` @ `202d73a` pushed. Tree clean.**
> Direct continuation of the guest_lists incident session (below) — owner said "move on to Full
> EditEvent architecture fix." Full detail: decisions log **v2.214–v2.216**; build phases **v2.83**.
>
> **PHASE 1 — diff-based Save (`EditEvent.jsx`):** re-verified the prior audit fresh (not just
> repeated old findings) and confirmed `roster_members`, `schedule_items`/`performances`/
> `rehearsals`, and `advance_items` all have a LIVE immediate-writer counterpart on EventDetails
> racing against EditEvent's batch-until-Save tabs — same shape as the guest_lists incident.
> `action_items` has no immediate-writer counterpart anywhere — narrower risk. Fix: `handleSave`
> now diffs `eventData` against the page-load-time `originalEvent` and only sends keys that
> actually changed, instead of the full record. `schedule_items` gets special handling (it's
> derived from performances/rehearsals — only regenerated/sent if any of the three changed).
> `guest_lists`/`roster_members` stay explicitly excluded regardless of diff outcome (atomic-
> committed elsewhere; re-sending even a "changed" local mirror risks re-triggering the bug).
> **Does NOT fully solve two people editing the SAME field concurrently** — only closes the far
> more common "editing one tab reverts an untouched field" case.
>
> **PHASE 2 — roster_members fully atomic:** the one field phase 1 alone couldn't protect (live
> same-field race: `PersonnelCard.jsx`/`AddPersonnelDialog.jsx` on EventDetails already wrote
> immediately, `PersonnelTab.jsx` on EditEvent batched only). New `rosterAdd`/`rosterUpdate`/
> `rosterRemove`/`rosterSetBandGroup` actions — same fresh-read → mutate → write → verify-retry
> shape as the guest_lists fix — keyed by `user_email` (roster entries have no stable id, unlike
> guests) instead of the old fragile array-index/name double-match `PersonnelCard.jsx` used.
> Converted all three writers. Inline field edits (role/instrument/phone/city) stay local-only on
> keystroke, commit atomically on blur.
>
> **⚠️ BASE44 50-FUNCTION CAP HIT AGAIN — now a CONFIRMED standing constraint, not a one-off:**
> attempted a genuinely new `manageEventRoster` function (unlike guest_lists' `mutateGuestList`,
> which had no prior deploy attempt to compare against) — failed identically: "Maximum of 50
> functions per app reached." **Every future new-domain fix will hit this too.** Folded into
> `decideGuestRequest` again (same `commitRosterWithRetry` pattern, same auth block reused).
> `decideGuestRequest` is now explicitly multi-domain (guest-list decisions/mutations + roster
> mutations) purely because of this slot constraint — documented plainly in its own file header
> so it doesn't read as an accident to a future session. **Assume any next architecture-fix phase
> (schedule_items/advance_items) will need the SAME consolidation trick from the start** — don't
> waste a deploy attempt creating a new function first; go straight to folding into an existing
> one, or ask the owner about the Base44 support ticket status on raising the limit.
>
> **Owner workflow this round:** implement → build/test → local checkpoint commit (no push) →
> present findings/open questions → owner said "test on localhost first" → then "deploy" →
> pushed + deployed both phases together. Consistent with the standing test-before-push-deploy
> rule; owner is comfortable batching a few local commits before one deploy round.
>
> **Still open, not started:** `schedule_items`/`performances`/`rehearsals` (`ScheduleTab.jsx`'s
> `updateScheduleItem`) and `advance_items` (`ActionItemsCard.jsx`) have the narrower
> "immediate-but-not-fresh-read, no retry" residual risk — same class as `decideGuestRequest`'s
> own un-upgraded `decide` action, not the worse batched-until-Save pattern already fixed.
> `action_items` — no immediate-writer counterpart, lowest priority. `EventDetails.jsx`'s
> `eventCacheRef` has no TTL (separate stale-display bug). Owner still needs to manually re-add
> Mark Whitfield Jr's 5 guests lost in the original incident (not a code task).

---

## 🔝 TOP BRIEF (2026-07-16 · superseded by the entry above, kept for detail)

> **LIVE = Serenova `0.7.0659` (deployed + verified). Repo `main` @ `51ae309` pushed. Tree clean.**
> Session started from a real production incident, ended with a new MobileHub feature. Full detail:
> decisions log **v2.209–v2.213**; build phases **v2.82**.
>
> **⚠️ CONFIRMED DATA-LOSS INCIDENT, ROOT-CAUSED + FIXED:** a band member (Mark Whitfield Jr) submitted
> 5 guest requests from MobileHub — his own notifications/emails proved the writes fired server-side,
> but none appeared in the Guest List. Root cause: `GuestManagerDialog`/`GuestListTab`'s Save (and
> mobile's `addGuestDirect`/`requestGuestManaged`) all batched edits locally and wrote the WHOLE
> `guest_lists` array back — a stale snapshot's Save silently erased requests that arrived after
> page-load. `submitStagedGuestRequest` also had an unguarded read-modify-write race (could let rapid
> concurrent submissions clobber each other). **Mark's 5 guests are very likely permanently lost** (no
> revision history in this data model) — owner re-adding manually from his emails, not a code task.
>
> **FIX — every `guest_lists` mutation is now atomic** (fresh-read → single-item write →
> verify-after-write retry, bounded — Base44 has no native optimistic concurrency): new action
> dispatch folded into **`decideGuestRequest`** (add/updateFields/remove/setComps/setCompsAll/
> setAssignee/decide) — **NOT a separate `mutateGuestList` function**, because deploying one hit
> Base44's **50-function-per-app hard cap** mid-session; consolidated into the existing function
> instead (zero new slots). `submitStagedGuestRequest` got `remove`/`lowerPartySize` actions too (same
> retry pattern) for a requester's self-manage. `GuestManagerDialog`'s Save button is **gone** — nothing
> left to batch. `EditEvent.jsx` no longer resends `guest_lists` in its global Save (stripped from that
> payload — it's exclusively atomic-committed elsewhere now).
>
> **⚠️ THE SAME ANTI-PATTERN STILL EXISTS ELSEWHERE — NOT YET FIXED:** `EditEvent.jsx`'s global "Save
> Changes" still batches-then-overwrites `roster_members`/`schedule_items`/`advance_items`/
> `action_items`/`additional_provisions`/`buyouts`/`contract_files`/`tour_manager_user_emails` (+ cross-
> event `Venue.contacts`) exactly the same way guest_lists used to. **This is the next task the owner
> asked for** ("move on to Full EditEvent architecture fix") — was starting this when this handoff was
> written. Same fix shape as guest_lists: atomic per-field/per-item server actions instead of one
> whole-record Save, likely folded into an existing function again given the 50-function cap.
>
> **Follow-ups shipped same session:** ticket-count number spinner → capped 1-10 dropdown (mobile +
> web, native spinners are fiddly to tap on phone); requester self-manage (remove own guest anytime incl.
> Confirmed, gated by a confirm popup; lower — never raise — own guest's ticket count, no fresh approval
> needed); fixed an "Add guest" 400 (the merged action wrongly required a name up front — GuestListTab's
> Add button deliberately creates a blank-name row, fills in during edit mode); linked companies
> (AMC/BMC/booking/tour mgmt) now assignable in "Assigned to" even with no email on file — tracking-only,
> tagged "(no email)", never triggers a notification.
>
> **📱 NEW MOBILEHUB FEATURE (unrelated to the incident, owner-requested):** the Home screen's "This
> Week" strip is now **"Next 7 Days"** — rolling yesterday/today/+5, not a fixed Sun-Sat week. Day cells
> are tappable → new **`MobileDayDetailDrawer`** (flight/hotel-style bottom sheet, Event/Performances/
> Rehearsals/Travel/Schedule grouping, same as the web's `DayDetailsDialog`). New "View calendar" header
> link → new **`MobileCalendarView`** full-screen month grid (mobile rebuild of `DailyCalendar.jsx`'s
> look, not a reused import — that one's web-only styled). New shared **`useMobileDayItemsMap`** hook
> builds the per-day items map once for all three surfaces. **Known gap, logged not fixed:**
> `MobileEventDetail` has no initial-date prop, so "Go To Event" from the popup for a non-today date
> lands on the event but not necessarily that date's itinerary sub-tab.
>
> **⚠️ WORKFLOW REMINDER (carried forward, still true):** build green ≠ tested. Confirm exact file/change
> list before pushing; deploy only on explicit "deploy"/"push live" go-ahead (this session got that
> go-ahead 4 times, each for a specific change). Base44 deploy recipe unchanged: `mv base44/entities
> ../_entities_HOLD` → `base44 functions deploy <name>` / `base44 site deploy -y` → move entities back.
> Verify via `curl app.serenovahub.com` bundle hash + grep the version string.

---

## 🔝 TOP BRIEF (2026-07-15 · superseded by the entry above, kept for detail)

> **LIVE = Serenova `0.7.0646` (deployed + verified). System Hub `1.0.0021`. Repo `main` @ `b555aa9` pushed. Tree clean.**
> Huge multi-arc session (2026-07-14 → 07-15). Full detail: decisions log **v2.199** (many entries this session). Highlights:
>
> **⚠️ WORKFLOW RULE (owner, reinforced — saved to auto-memory `test-before-push-deploy`):** TEST on `localhost:5173`
> BEFORE commit/push/deploy. A green build is NOT a passing test — the owner found many real runtime bugs by hands-on
> testing this session. Flow: change → build passes → say "ready to test on localhost:5173" → wait → only deploy on
> the owner's explicit "ship it". Do NOT auto-deploy on green.
>
> **🎫 GUEST LIST (all live):** export redesign (separate CSV/PDF buttons), **titled PDF** (By Show/By Date, keep-a-show-
> together page-breaks, running header + page numbers, "Created by Serenova Hub" footer), **public shareable Guest List
> link** (`/GuestList/:code`) — dedicated data-minimal `SharedGuestList` page; **Share Guest List dialog** (own Share-menu
> item) with recipient multi-select (band/crew/venue) + **email send** via `@/integrations/Core` SendEmail (migrate to our
> own email service off-Base44 later); landing page **Download (CSV/PDF)** with "as of" stamp. **Smart short links** —
> base62 8-char codes, clean paths `/GuestList/:code` + `/Itinerary/:code` (legacy `?share_token=` still works), existing-
> link reuse in both share dialogs. **⏳ OWNER:** add `expires_at` (date-time) to the **ShareableLink** entity in the Base44
> editor so guest-list links carry the 14-day expiry (functions `generateShareableLink`/`getSharedItineraryData` already
> deployed with expiry logic).
>
> **🛠️ SYSTEMHUB:** new **Staged Users** browser (tab + clickable KPI; `adminStagedSessions.list_all` deployed). Root-caused
> a stale **`functions_version` localStorage pin** (was masking new function deploys) — guarded in `src/lib/app-params.js`.
>
> **📱 MOBILEHUB event detail:** Contacts now includes **Tour Manager(s) + Associated Crew** (phone resolved from
> Membership `phone_mobile`); right-aligned tap-to-call/email CTAs; **venue route map + capacity/Database badges** ported
> from web; **date pills moved into the Schedule tab** (thinner header); guest order + Venue-tab parking crash fixed.
> **Known gap:** a cross-account (management) TM's phone doesn't resolve (lives on their mgmt-account membership).
>
> **🎼 SET LISTS (big):** page **rebuilt** as sortable/filterable table (Date·Venue·Band·Set List·Songs, Event/band join,
> click→editor, row delete). **Analytics** = its OWN nav page (Repertoire → **Analytics / Set Lists / My Repertoire**) with
> KPIs, top songs, dormant/never-played, match-rate, **PDF report**. **⭐ SET LIST TEMPLATES rebuilt SUPABASE-FIRST**
> (owner: cleanest + off-Base44): new Supabase tables **`set_list_templates`** + **`set_list_template_usages`** behind seam
> **`src/api/setlistTemplates.js`** (temp `anon` RLS, settlements pattern), standalone **New Template editor** (event-less,
> band-assigned, SongTypeahead), **Save-as-Template → Supabase** (no more Historical dup — that bug was Base44 dropping the
> non-existent `is_template` field), **provenance/usage logging** on save (Autofill→From template), **Template usage +
> Unused** analytics panels. NOT on the Base44 SetList entity by design.
>
> **DEPLOY:** unchanged — `git push` + `mv base44/entities aside → npx -y base44@0.0.56 --app-id 68c22e8ff3726c063c4a53e2
> site deploy -y → restore`. Functions deploy same but with `functions deploy <names>`. Build: `BASE44_LEGACY_SDK_IMPORTS=true
> ./node_modules/.bin/vite build`. Bump `src/version.js` first. **AUTHORSHIP: Adam Jones only, no Claude trailer.**
>
> **NEXT / BACKLOG:** header/template spec (Phase HD, not started — inventory done: 4 header heights, no shared component,
> chrome bar is ours in `Layout.jsx:1173`); template follow-ups ("Based on <template>" badge, template usage in the PDF,
> fuzzy typo de-dup); the two owner Base44 items above (`expires_at` field; cross-account TM phone).

---

## 🔝 PRIOR TOP BRIEF (2026-07-14 evening · superseded — the 4 to-do items here are all DONE, see the 2026-07-15 brief above)

> **LIVE = Serenova `0.7.0629` (deployed + verified — grepped the version string in the live bundle
> at `app.serenovahub.com`). Repo `main` @ `18ffc17`, pushed to GitHub. Desktop session done for the
> day; handing off to LAPTOP.** Decisions log **v2.184**, build phases **v2.79**, CLAUDE.md refreshed.
>
> **This was a long hands-on-testing session on EventFinancialDetails + the Guest List popup —
> shipped in three deploys (`0.7.0627` → `0.7.0628` → `0.7.0629`), owner testing and iterating live
> the whole way, not a code-review pass.**
>
> **`0.7.0627` — EventFinancialDetails test-and-fix pass.** `CommissionOverrideManager` was silently
> rendering nothing (wrong props passed — now 3 instances, one per commission slot, overrides persist);
> duplicated total formulas had drifted the top summary bar out of sync with section badges (consolidated
> to one shared source); **Business Manager commission is account-level, not per-event** (owner correction
> — built a per-event dropdown first, reverted it; both commission calculators now resolve the account's
> Business Management company directly, no event field needed); summary bar now floats fixed-to-viewport
> on scroll (was unreliable `position: sticky`) showing all 7 deduction buckets; two Edit dialogs (Team
> Payments, Admin Fees) saved but never closed (no-op `onClose`, fixed); numeric `$` inputs across 5
> files/11 spots snapped back to "0" on every keystroke that emptied them — new shared
> `src/components/financial/NumberInput.jsx` fixes all of them (**same bug still open in the Settings
> page's `TeamRateSettings.jsx`/`FinancialDefaults.jsx` — logged, not fixed, different page never
> touched this session**); accordion sections merged into single bordered cards; Commissions +
> Settlement Checklist default open. **New:** Admin Fee "Fee Name" is a searchable/free-type field
> (native datalist) seeded with 7 defaults, learns new names per-account via a new
> `AccountFinancialDefaults.admin_fee_names` field (**caveat: untested against a cold Base44 schema
> cache — watch for silent persistence failures if new names stop sticking**).
>
> **`0.7.0628` — Guest List popup ("Manage Guest List") redesign.** Sticky header/footer (Export
> moved into the header via a state-backed DOM-node portal — a plain `useRef` doesn't work for this,
> refs attach after the child's first render); "Comps per show" bulk setter; per-show teal header
> consolidated ("Comps 3 / [10] used" instead of two redundant numbers); redundant "Confirmed" badge
> removed; new "Assigned to" roster picker (routes through the existing `decideGuestRequest`
> approve-pipeline — no new notify backend); real names resolved via `getUsersByEmailsForAccount` +
> `Membership` (same fix Personnel/Travel already had); darker card shading; widened two number inputs
> + new `.spin-light` CSS util for visible spinner arrows on dark backgrounds.
>
> **`0.7.0629` — "Assigned to" picker fixes (root-caused via an owner-authorized live read-only
> check) + row redesign.** Confirmed via `base44 exec` against production (read-only, narrowly scoped,
> owner explicitly authorized after the auto-mode classifier blocked the first unauthorized attempt):
> (1) TM push/email notification is **NOT broken** — `adam@originalartists.com`'s TM assignment +
> active PushSubscription are both correctly configured; the real explanation is that every guest on
> the test event was added via the admin dialog, which never calls `submitStagedGuestRequest` (the
> only function that notifies a TM of a "new request") — nothing to fix, just no real mobile-submitted
> request had happened yet. (2) Crew + the event's Tour Manager(s) were missing from "Assigned to" —
> `roster_members` alone doesn't carry them; now folds in `tour_manager_user_email(s)` + every roster
> member's Membership `associated_crew` (found the exact case owner described: a band member's
> personal Tour Manager). (3) Artist-first sort was checking `Membership.role === 'owner'` — wrong;
> the artist's real role is `'artist'`, confirmed live — now checks `Account.owner_user_email`/
> `artist_user_email` (same signal `decideGuestRequest` already uses). **Row redesign:** permanent
> `added_by_email`/`added_by_name` stamped on every guest at creation (answers "who added this guest,"
> separate from "who it's assigned to"); view-mode-by-default (plain text, no dropdown/delete/confirm
> clutter); new pencil icon opens ONE row into edit mode at a time; Requested (pending) guests always
> show quick ✓/✗ regardless of edit state; brand-new guests open directly into edit mode.
>
> **Also answered (no code, just findings):** owner asked whether the teal/tan brand direction is
> centrally documented — **logged as Phase TH (Global Theme Centralization), scoped, NOT STARTED.**
> `docs/MOBILEHUB-WORKING-DOC.md` has the real owner-approved palette (teal `#284854`/tan `#d9a774`
> accent-only/cream `#faf8f4`/sage `#b8b8aa`), explicitly never rolled out past MobileHub. The system
> default today is still the BLUE `--harmony-*` CSS vars in `Layout.jsx`. `src/config/hubThemes.js`
> already has the correct teal/tan values and a `hubTheme()` accessor built for exactly this — **zero
> actual usages anywhere**, every one of ~39 components pastes the hex literal instead. See decisions
> log 2026-07-14T18:00 EDT for the full scope-when-picked-up plan.
>
> ## 👉 UNFINISHED — laptop pick-up, in priority order (owner's own words)
>
> 1. **Guest List Export dropdown redesign** — change "Export All" from one full-row button to
>    `Export All   [PDF] [CSV]` with separate icon buttons on the right (same for each per-show row
>    in the dropdown). File: `src/components/events/edit/GuestListTab.jsx` (the `exportMenu` /
>    `downloadCsv` section).
> 2. **Clean, titled PDF export** — needs an actual visual design pass (title, clean layout), with
>    **"By Show"** (current per-performance grouping) AND **"By Date"** (owner-confirmed meaning: a
>    same-day rollup merging multiple shows on one calendar date — owner picked the recommended
>    option for this). **No PDF generation exists yet for guest lists** — CSV only today. Reuse the
>    jsPDF pattern already proven in `base44/functions/generateFinancialSummaryPdf/entry.ts` (server-
>    side `new jsPDF({format:'letter'})`) — don't reinvent as client-side jsPDF. Will need either a
>    NEW cloud function (mind the 50-new-function cap — 174 existing are grandfathered) or repurpose
>    an existing one (established convention: `diagnoseManagementAccess`/`syncTeamPaymentsForEvent`
>    were both repurposed to dodge the cap).
> 3. **Public shareable Guest List link** (e.g. to send to a venue) — clean, legible, presentable,
>    with its own PDF/CSV download (All or by Date) built in. Model on the existing pattern:
>    `ShareableLink` entity (`base44/entities/ShareableLink.jsonc` — `link_type` enum needs a new
>    value, e.g. `event_guest_list`), `generateShareableLink`/`validateShareableLink` cloud fns
>    (generic enough to reuse as-is), `getSharedItineraryData` (extend with a new branch, or copy its
>    pattern into a new fn), a new public route mirroring `src/pages/SharedEventView.jsx` (registered
>    in `src/App.jsx` — extend the `isSharedItinerary`-style auth-bypass check to also match the new
>    route), and the share-dialog UX mirrored on `src/components/mobile/MobileShareDialog.jsx`.
>    **Expiry rule (owner decision, needs building — doesn't exist for ANY share link today):**
>    final performance/event date + 14 days. **Owner explicitly wants this retrofitted onto the
>    EXISTING itinerary share links too**, not just the new guest-list one — compute dynamically at
>    validation time (don't store a redundant `expires_at`; derive from the linked event/tour's own
>    last performance date so it stays correct even if dates change later). Skip this expiry logic
>    for `link_type: 'staged_session'` — that already has its own unrelated 30-day idle TTL.
> 4. **SystemHub "Staged Users" browser UI** — backend is DONE (`adminStagedSessions` cloud fn gained
>    a `list_all` action, already deployed: every staged identity platform-wide, grouped by email,
>    `is_active`/`active_sessions`/`total_sessions`/`last_active`/`devices` computed, real names via
>    `User.list()`). **No UI built yet.** Owner wants: (a) the Overview "Active Staged" KPI tile
>    (`src/components/systemhub/HubOverview.jsx` — currently a static, non-clickable `Card`) made
>    clickable → jumps to a new "Staged Users" sub-tab; (b) that sub-tab as a THIRD tab alongside
>    "All Users"/"System Users" in `src/pages/SystemHub.jsx`'s Users tab (`usersSubTab` state already
>    exists — mirror the `setUsersSubTab('all')` + `setActiveTab('users')` pattern used elsewhere in
>    that file for deep-linking); (c) a filter toggle (All / Active / Inactive) and a table (name/
>    email, active vs total sessions, last active, device labels). Owner ALSO separately asked
>    (unresolved, worth a follow-up conversation not just a code change): is requiring a 2-character
>    search before "All Users" shows anything actually more secure, or just an arbitrary UX choice?
>    (Researched: **not security** — `systemUsersSearch`'s backend loads ALL users/accounts into
>    memory regardless of query length once past the 2-char gate; the gate is purely client-side UX,
>    no stated rationale in code. "System Users" shows its list with zero search needed, same
>    permission boundary.) Owner also wants staged users' active/inactive status visible directly in
>    the "All Users" search results, not just after opening the detail panel.
>
> **Lower priority / just logged, not scoped further:** the `NumberInput` snap-to-zero bug also
> exists in `src/components/financial/TeamRateSettings.jsx` and `FinancialDefaults.jsx` (Settings
> page) — same fix (`src/components/financial/NumberInput.jsx` already exists, just needs wiring in).
>
> **DEPLOY NOTE (unchanged):** Claude executes the Base44 site deploy directly after git push + owner
> OK. Build: `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`. Deploy:
> `mv base44/entities base44/.entities-hidden && npx -y base44@0.0.56 --app-id
> 68c22e8ff3726c063c4a53e2 site deploy -y && mv base44/.entities-hidden base44/entities`. Always
> bump `src/version.js` (`HUB_VERSIONS.serenova`) FIRST, build, verify the version string landed in
> `dist/assets/*.js`, commit, push, deploy, then verify live via `curl` against the deployed bundle.
>
> **LOCAL PREVIEW:** `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite --port 5173 --host
> 0.0.0.0` (LAN-accessible; real auth + live data via the `/api` proxy — nothing here is a mock).
>
> **AUTHORSHIP:** commits are **Adam Jones only — NO Claude trailer.**
---

## 🔝 PRIOR TOP BRIEF (2026-07-01 · superseded)

> **LIVE = Serenova `0.7.0626` (deployed + verified). Repo `main` @ `b15cb22` pushed to GitHub.**
> Session focus: **Expenses feature — fully built and live.**
>
> **💸 EXPENSES — COMPLETE & LIVE (`0.7.0624`→`0.7.0626`).** The Event Expenses & Reimbursements dialog on
> EventDetails → Data Room tab is now fully functional:
> - **Log expense** — manual form OR upload receipt → AI auto-fill (type/amount/date/description via `ExtractDataFromUploadedFile`). Reimbursement checkbox → assign `reimbursed_to_email` from event roster.
> - **List view** — grouped by `expense_type` (EXPENSE_TYPES canonical order), compact single-row per item, category subtotals, amount, status pill (reimbursement only), receipt icon, trash (hover, confirm-in-place).
> - **Delete** — routes through `diagnoseManagementAccess` cloud fn with `action:'delete'` + `expenseId`. Privileged roles (`owner/super_admin/admin/tour_manager`) can delete any; others delete own only. Concurrent-delete guard: `if (deletingId) return`.
> - **Reads** — also through `diagnoseManagementAccess` (`asServiceRole.entities.Expense.filter`). This bypasses Base44 entity read RLS which blocks user-level reads. Entity RLS fix deferred (logged v2.173; can't `entities push` due to `user_condition` validator; would need Base44 admin console edit).
> - **`ExpensesCard`** on EventDetails shows per-category breakdown rows + total.
> - **Key files:** `src/components/events/details/ExpensesCard.jsx`, `src/components/events/details/ExpensesDialog.jsx`, `base44/functions/diagnoseManagementAccess/entry.ts`.
>
> **⚠️ BASE44 FUNCTION LIMIT:** 50 new functions hard cap per project (existing 174 grandfathered on Elite plan). `diagnoseManagementAccess` was repurposed (name kept) to avoid creating a new function. Owner has Base44 support contact open. When limit is lifted: rename to `getExpensesForEvent`, restore original diagnostic fn.
>
> **👉 FINANCE NEXT (owner-flagged):**
> (1) **`EventFinancialDetails`** — integrate expense data from the cloud function (not `Expense.filter()` directly); update the expense display there to match the new grouped layout.
> (2) **FinancialHub "Event Finances" cards** — point at shared calculators (`eventFinanceSummary.js` / `eventCommissions.js`); currently show stale Team $1,900 / Expenses $0.
> (3) **Member-level Pay & Rates** section on Team member dialog (finance-gated).
> (4) **Travel-day** rate component (manual per event).
> (5) **Summary / Close-out + Closeout Wizard**.
>
> **🔗 PHASE ML (Cross-Artist Member Linking) — SCOPED, PENDING OWNER BACKEND DEPLOYS** (same as before — Finances don't depend on it).
>
> **DEPLOY NOTE:** Claude executes Base44 site deploy directly (git push + owner OK). Functions deploy also works. Build cmd: `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`. Always bump version FIRST in `src/version.js` (HUB_VERSIONS.serenova), then build, verify version in dist, commit, push, deploy.
>
> **LOCAL PREVIEW:** `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite --port 5173 --host 0.0.0.0`
>
> **AUTHORSHIP:** commits are **Adam Jones only — NO Claude trailer**.

## 🔝 PRIOR TOP BRIEF (2026-06-26 PM · superseded)

> **LIVE = Serenova `0.7.0610` (deployed + verified). Repo `main` @ `b812347` pushed to GitHub.**
> Huge multi-part session. Two big arcs: (A) **Team/Member management** + (B) **Finances web-first** (current focus).
>
> **🆕 LOCAL PREVIEW is now `--port 5173 --host 0.0.0.0`** (LAN-accessible so the **laptop** can test the
> desktop preview at `http://<desktop-LAN-IP>:5173`, e.g. `192.168.7.198`). Owner is **switching to the
> laptop** — `git pull` there, `npm install`, run the preview command. Two-machine sync via git (push from
> wherever you work, pull on the other). See `SERENOVA-MACHINE-SYNC.md`.
>
> **💰 FINANCES (web-first) — current focus, deployed `0.7.0610`.** Built on the existing finance
> foundation (FinancialHub + EventFinancialDetails + EventFinancials). Shipped: **team payments AUTO-SYNC
> to the event roster** (NO button — pay line from `AccountFinancialDefaults` rate defaults × **gig days**
> = distinct `event.performances` dates; manual edits are `is_override` and protected, never reverted);
> an **at-a-glance finance strip on EventDetails** (top, gated by `financials` view) showing **Gross ·
> Band & Crew · Other Exp · Commissions · Buyout · Est. Net**, **LIVE-computed from source** (no need to
> open the settlement page) — flights/hotels pulled from `FlightBookingGroup.total_cost`/`Accommodation
> .total_cost`; a **shared commission calculator** (`src/utils/eventCommissions.js`) + finance-summary util
> (`src/utils/eventFinanceSummary.js`). **Fixed a pre-existing blank `EventFinancialDetails` crash**
> (`CommissionOverrideManager` undefined-commission guard). Net = Gross − Band&Crew − Other − Commissions
> + Buyout (buyout POSITIVE). Source of truth: decisions log **v2.164**, build phases **v2.71**,
> `docs/FINANCIALHUB-WORKING-DOC.md`.
>
> **👉 FINANCE NEXT (owner-flagged):** (1) **FinancialHub "Event Finances" cards** still use the old calc
> (Team $1,900 stale / Expenses $0) → point them at the shared calculators; (2) **member-level Pay & Rates**
> section on the Team member dialog (finance-gated) + keep the Finance Hub bulk table (one source of truth);
> (3) **travel-day** rate component (owner chose **manual per event**); (4) **Summary / Close-out + Closeout
> Wizard**; (5) DRY-merge EventFinancialDetails' commission math onto the shared util. Rate model = per-gig
> (flat) · per-day (× gig days) · travel-day (smaller, manual) · per-diem (× days).
>
> **🔗 PHASE ML (Cross-Artist Member Linking & Consent) — SCOPED, partly built, NOT fully live.**
> Spec: `docs/MEMBER-LINKING-CONSENT-WORKING-DOC.md`. Built: roles/positions rework (profile-as-key,
> Position + Role Description flow-through), Band Groups fixes, **Associated Crew**, two-step **"Add or
> Invite to Team"** (team-member vs **associated-with** fork), **member removal** (forward-only). **⏳ OWNER
> -SIDE BACKEND DEPLOYS PENDING** for these to persist/work live: (a) add Membership fields
> `associated_crew` (array) + `removed_at` (date-time) in the **Base44 entity editor**; (b) deploy the
> `lookupPersonByContact` function (`! mv base44/entities base44/.entities-hidden; npx -y base44@0.0.56
> --app-id 68c22e8ff3726c063c4a53e2 functions deploy lookupPersonByContact; mv base44/.entities-hidden
> base44/entities` — the harness blocks Claude from the functions-deploy bypass; owner runs it or adds a
> Bash permission rule). **Finances DON'T depend on these.** Big design logged: associated crew DERIVED
> from the management relationship (auto-scope, Phase 0/F-era); LIVE access model; data-ownership boundary.
>
> **DEPLOY NOTE:** Claude now executes Base44 **site deploy** directly after git push + owner OK (proven;
> `mv base44/entities aside` → `site deploy -y` → restore). `functions deploy` bypass is harness-blocked.
> **AUTHORSHIP:** commits are **Adam Jones only — NO Claude trailer**.

> **LIVE = Serenova `0.7.0610` (deployed + verified). Repo `main` @ `b812347` pushed to GitHub.**
> Huge multi-part session. Two big arcs: (A) **Team/Member management** + (B) **Finances web-first** (current focus).
>
> **🆕 LOCAL PREVIEW is now `--port 5173 --host 0.0.0.0`** (LAN-accessible so the **laptop** can test the
> desktop preview at `http://<desktop-LAN-IP>:5173`, e.g. `192.168.7.198`). Owner is **switching to the
> laptop** — `git pull` there, `npm install`, run the preview command. Two-machine sync via git (push from
> wherever you work, pull on the other). See `SERENOVA-MACHINE-SYNC.md`.
>
> **💰 FINANCES (web-first) — current focus, deployed `0.7.0610`.** Built on the existing finance
> foundation (FinancialHub + EventFinancialDetails + EventFinancials). Shipped: **team payments AUTO-SYNC
> to the event roster** (NO button — pay line from `AccountFinancialDefaults` rate defaults × **gig days**
> = distinct `event.performances` dates; manual edits are `is_override` and protected, never reverted);
> an **at-a-glance finance strip on EventDetails** (top, gated by `financials` view) showing **Gross ·
> Band & Crew · Other Exp · Commissions · Buyout · Est. Net**, **LIVE-computed from source** (no need to
> open the settlement page) — flights/hotels pulled from `FlightBookingGroup.total_cost`/`Accommodation
> .total_cost`; a **shared commission calculator** (`src/utils/eventCommissions.js`) + finance-summary util
> (`src/utils/eventFinanceSummary.js`). **Fixed a pre-existing blank `EventFinancialDetails` crash**
> (`CommissionOverrideManager` undefined-commission guard). Net = Gross − Band&Crew − Other − Commissions
> + Buyout (buyout POSITIVE). Source of truth: decisions log **v2.164**, build phases **v2.71**,
> `docs/FINANCIALHUB-WORKING-DOC.md`.
>
> **👉 FINANCE NEXT (owner-flagged):** (1) **FinancialHub "Event Finances" cards** still use the old calc
> (Team $1,900 stale / Expenses $0) → point them at the shared calculators; (2) **member-level Pay & Rates**
> section on the Team member dialog (finance-gated) + keep the Finance Hub bulk table (one source of truth);
> (3) **travel-day** rate component (owner chose **manual per event**); (4) **Summary / Close-out + Closeout
> Wizard**; (5) DRY-merge EventFinancialDetails' commission math onto the shared util. Rate model = per-gig
> (flat) · per-day (× gig days) · travel-day (smaller, manual) · per-diem (× days).
>
> **🔗 PHASE ML (Cross-Artist Member Linking & Consent) — SCOPED, partly built, NOT fully live.**
> Spec: `docs/MEMBER-LINKING-CONSENT-WORKING-DOC.md`. Built: roles/positions rework (profile-as-key,
> Position + Role Description flow-through), Band Groups fixes, **Associated Crew**, two-step **"Add or
> Invite to Team"** (team-member vs **associated-with** fork), **member removal** (forward-only). **⏳ OWNER
> -SIDE BACKEND DEPLOYS PENDING** for these to persist/work live: (a) add Membership fields
> `associated_crew` (array) + `removed_at` (date-time) in the **Base44 entity editor**; (b) deploy the
> `lookupPersonByContact` function (`! mv base44/entities base44/.entities-hidden; npx -y base44@0.0.56
> --app-id 68c22e8ff3726c063c4a53e2 functions deploy lookupPersonByContact; mv base44/.entities-hidden
> base44/entities` — the harness blocks Claude from the functions-deploy bypass; owner runs it or adds a
> Bash permission rule). **Finances DON'T depend on these.** Big design logged: associated crew DERIVED
> from the management relationship (auto-scope, Phase 0/F-era); LIVE access model; data-ownership boundary.
>
> **DEPLOY NOTE:** Claude now executes Base44 **site deploy** directly after git push + owner OK (proven;
> `mv base44/entities aside` → `site deploy -y` → restore). `functions deploy` bypass is harness-blocked.
> **AUTHORSHIP:** commits are **Adam Jones only — NO Claude trailer**.

## 🔝 PRIOR TOP BRIEF (2026-06-24 LATE PM · superseded)

> **Repo `main` @ `a093414` = `0.7.0608`, build-green, pushed. NOT deployed (LIVE still `0.6.0604`).**
> Two EventDetails bugs the owner found in local preview are FIXED + verified in preview; a standing
> **prep-on-touch** migration policy is now in effect. Docs: decisions log **v2.160**, CLAUDE.md updated.
>
> **🐞 Bug 2 (FIXED) — Personnel showed email prefixes ("grandbaton") not names.** `PersonnelCard` now
> resolves band/crew names via `getEventTravelSummary` (`name_on_ticket`), matching the Travel tab.
> Prep-on-touch: swapped its legacy `usePermissions` → canonical `usePermissionContext()`.
> **🐞 Bug 3 (FIXED — live DB) — Accommodations card vanished for non-owner admin/TM.** `role_templates.
> perm_accommodations` was `none` for all privileged artist roles; owner ran SQL to seed it = `perm_travel`
> (super_admin/admin/tour_manager → `full`). No code change. **Both verified good in preview by owner.**
>
> **🆕 PREP-ON-TOUCH POLICY (owner-approved):** as we edit files, also bring THEM to migration-ready
> patterns (canonical perms; `base44.entities.*` not `@/entities/*`; data behind `src/api/<domain>.js`
> seams) — bounded relaxation of scope-only rule #3. Bigger refactors are LOGGED, not done. Full detail:
> CLAUDE.md → "Migration-readiness (prep-on-touch)" + decisions log v2.160. **Caveat:** makes the Base44
> exit cheaper, does NOT unblock it — **Phase 0/F** (Supabase auth session + RLS) is still the gate.
>
> **AUTHORSHIP REMINDER:** commits are **Adam Jones only — NO Claude trailer** (laptop slipped once this
> session and amended+force-pushed `b393d02`→`a093414` to remove it; clean now).
>
> **👉 NEXT (desktop):** (1) **role-reassignment walkthrough + team updates** (owner flagged on Personnel
> role editing — likely ties to the placed-membership model); (2) **MAJOR: Finances — web first**, then
> MobileHub. **Deploy `0.7.0608` when ready** (owner-run `base44 site deploy` + entities move-aside +
> curl/canary verify). **Logged prep follow-ups:** `AccommodationCard` legacy `@/entities/Accommodation`
> + legacy perms; create `src/api/events.js`/`travel.js` seams (structural task).

## 🔝 PRIOR TOP BRIEF (2026-06-24 PM · superseded)

> **Events (web) UX + brand pass DONE — repo `0.7.0607`, build-green, pushed to `main`. NOT deployed
> (live still `0.6.0604`).** Owner reviewed via a new **local preview** and is happy; refinements applied.
> Docs: decisions log **v2.159**, build phases **v2.69**.
>
> **🔍 NEW — LOCAL PREVIEW workflow (test before live, no deploy):** plain Vite + the Base44 `/api`
> proxy to the live backend. `git pull` → `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite --port 5173`
> → `localhost:5173` → log in (real auth + LIVE data) → serves the latest local commit. Committed
> **`.env.development`** carries the public params so it works on any machine. Base44 has NO CLI
> deploy-to-preview; `base44 dev` is unusable (needs Deno + strips entities). **Full recipe:
> `SERENOVA-MACHINE-SYNC.md` → 🔍 LOCAL PREVIEW.**
>
> **What shipped this pass (`0.7.0605`→`0.7.0607`, web Events area):** Events landing teal/tan brand
> header + status-colored card borders + bordered tab control + slimmer cards + shows→count +
> Action-Required inline pill; Event Details **People** (Management 3-col, Personnel real First-Last
> names via `getUsersByEmailsForAccount`, real account role, thinner rows, Cancel-right, full role list);
> **Travel** (Accommodations above Travel + clean 3-col location·dates·people; TravelCard date-led/
> time-sorted slim rows + SWR cache); **Accommodations popup** (slim rooming + cached Getting-Around
> distances). `syncFromGithub` fn pulled into repo for reference. Auth/access verified canonical.
>
> **👉 NEXT (owner, on laptop):** (1) **role-reassignment walkthrough** + **team updates** (owner flagged
> "we need to go through that" on Personnel role editing); (2) continue Events tweaks; (3) **MAJOR:
> Finances** — web first, then MobileHub. **Deploy `0.7.0607` to live when ready** (owner-run
> `base44 site deploy`, then curl/canary verify). **Open follow-ups:** global teal retheme (harmony
> tokens in `Layout.jsx`); repo authorship credit line; route-info needs `getVenueRouteInfo` redeploy.

## 🔝 PRIOR TOP BRIEF (2026-06-24 AM · superseded)

> **🏁 MILESTONE — current MobileHub build COMPLETE.** **Live = Serenova `0.6.0604`** (bundle `index-HQn2KYdL.js`); **repo is bumped to `0.7.0605`** (`main` @ `d11c707`) — the **0.6→0.7 minor bump marks the milestone** and ships on the NEXT deploy (live still shows `0.6.0604` until then). Build green via `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`.
> Docs: decisions log **v2.157**, build phases **v2.67**. Backup tag **`safepoint-0604`** on the remote (prior `safepoint-0587`).
> **Versioning policy (owner-confirmed):** `minor` = the app's KEY development milestones; `build` is a CONTINUOUS per-hub counter that NEVER resets across minor bumps (`0.6.0604`→`0.7.0605`) — shows uninterrupted development + stays a strictly-increasing deploy canary. (Codified in `src/version.js` header.)
> **👉 NEXT FOCUS (owner, this hub only):** (1) back to the **Events pages** for minor fixes; (2) **MAJOR: Finances** — build on **web FIRST**, then bring into **MobileHub only AFTER** the web Finances reaches a good milestone. MobileHub itself is feature-complete + stable for now.

**`0.6.0604` (final this session):** guest-decision notice **dismiss is now instant** (local `dismissedNoticeIds` + background `dismissStagedNotice`) — no full-screen reload, header/scroll stay put. **Flight "boarding" researched + deliberately NOT built** (AeroAPI `/flights/{ident}` exposes no boarding time / "Now Boarding" — airline-proprietary; owner: "leave it off, what we have is great"; `estimated_out` is the closest real signal if ever wanted).

**✅ DONE 2026-06-24 — MobileHub UX run `0.6.0593`→`0.6.0603` (all live, build-green). Full detail: decisions log v2.155.**
- **Live Flight bar/sheet:** docked to the nav (no gap, status-tinted, rises from menu); expanded sheet = gradient status hero + countdown, status-tinted route card, **plane progresses along the route line** (live in-flight), bolder Gate & Terminal.
- **Add to Calendar** (shared `src/utils/calendarExport.js`): flight + ground `.ics`; notes = own CONF only, airline, times, **travellers as a line-broken name list**, **2h/3h check-in** event (by `route_type`); floating airport-tz times; **flight travellers grouped across per-PNR flight records** (TM sees all). **Confirmed working.**
- **Pull-to-close + scroll-lock** on all sheets (shared `src/hooks/useSheetDrag.js`); **hotel rooming list** compacted (name left / room info right).
- **Events tab:** Past = descending; fixed tour-header duplication/phantom-buttons (React key collision → unique keys).
- **Pull-to-refresh + reopen-refresh** (>5min idle AND online only); **silent reload** (`loadAccountData(account,{silent})`) keeps you on the current event+tab+scroll; discrete **"Refresh cache"** button in the hamburger menu (clears SW + Cache Storage, cache-busts).

**🟢 PERMISSIONS — mgmt-TM edit on MobileHub RESOLVED (`0.6.0600`).** A logged-in TM (owner of Original Artists, linked to Lisa as `tour_manager`) now has setlist/guest edit on mobile. A temp diagnostic proved the linked-account bridge resolves to **`admin`** correctly (seat owner→admin, event-scoped link + matching `ManagerArtistAssignment`); earlier view-only was a transient data state. **👉 Owner's architectural direction (next phase):** make management-linked users **"placed" memberships inside the artist account** — visible/read-only to the artist, managed only by the management company/dashboard — collapsing the cross-account bridge into the normal membership path. **Related gap:** the management-dashboard UI to assign a management↔artist role appears lost (Team-page role is locked/uneditable) — needs rebuilding (likely the same phase).

**👉 OPEN / NEXT:** (a) **placed-membership** model + management-dashboard role-assignment UI (above); (b) optional **in-app "update available → reload" prompt** for PWA users (no SW precache today — staleness is just `index.html`/HTTP); (c) gate the "Refresh cache" button behind a role/flag if it shouldn't be public; (d) earlier-session opens still stand (Push Group 3 webhook, entity-RLS-format migration so deploys skip the `base44/entities` move-aside, `@/entities/*`→`base44.entities.*` Phase 9, Calendar Integration Phase C).

---

## 🔝 PRIOR BRIEF (2026-06-23 PM)

> **Serenova `0.6.0592` LIVE** (deployed via `base44 site deploy`, verified — bundle `index-CkTLwMK_.js`) · `main` @ `1374e86` · build green via `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`.
> Docs: decisions log **v2.153**, build phases **v2.66**. Backup tag `safepoint-0587` on the remote.

**🔴 FIXED `0.6.0592` (commit `1374e86`, live) — CLI builds broke the LEGACY `@/entities/*` client + Events 42703 + EventDetails loop.** A TM (Adam @ Original Artists, in the Lisa Fischer account) hit `Base44Error: App not found` 404 + an infinite "Loading event…" spinner on EventDetails — while the Events LIST and MobileHub showed the event fine. **Root cause:** EventDetails reads via the *legacy* `@/entities/Event` compat client (`@base44/vite-plugin/compat/base44Client.cjs`, `serverUrl = process.env.VITE_BASE44_BACKEND_URL`, no origin fallback), separate from the canonical `base44.entities`. The vite plugin reads `mode` from the **wrong config-hook arg** → its `loadEnv` defaults to development → `process.env.VITE_BASE44_*` defined as `undefined` → bundle had `serverUrl:void 0` → SDK default host → "App not found" on **every** `@/entities/*` read (~60 files; the frozen-editor builds had masked it). **Fix:** `vite.config.js` re-injects the define with the correct mode via a plugin after `base44()` (bundle now bakes `serverUrl:"https://app.serenovahub.com"`). Also: `AllEvents.jsx` off the legacy `useMultiplePermissions` (it queried the non-existent `memberships.permission_template_id` → 42703, suppressing all event-card alert badges) → canonical `canView('events')`; `EventDetails.jsx` loop fixed (dropped `loading` from `loadEvent` deps) + clean "Event not found" state on 404. Decisions log v2.153 has the full chain. **Deploy gotcha confirmed:** `base44 site deploy` DOES validate `base44/entities` → the `mv base44/entities ../_entities_HOLD` move-aside is mandatory around every deploy (Account.jsonc `user_condition` RLS format). **Follow-ups (logged):** migrate the entity RLS format so deploys stop needing the move-aside; `base44 dev` still leaves the legacy client unconfigured in dev; migrate `@/entities/*`→`base44.entities.*` (Phase 9).

**🔴 DEPLOY CHANGED — the Base44 editor "Publish" is BROKEN; deploy via the `base44` CLI now.**
The editor ships a **frozen build** (served bundle hash never changes across "successful" publishes; live
stayed on stale `0584` for weeks while the editor showed new code) and its git auto-sync wedged (an
authorship force-push rewrote a commit it had ingested). Use the CLI (`npx -y base44@0.0.56`, app-id
`68c22e8ff3726c063c4a53e2`):
- `base44 login` (device-code — run in a REAL terminal; token persists to disk, shared with Claude's shell).
- **Site:** build → `base44 --app-id … site deploy -y` (ships `./dist`; production-direct, NO preview). **`-y` REQUIRED in a non-interactive shell.**
- **🔴 `.env.production` MUST exist (committed) or every `/api` call 404s.** A plain `vite build` omits Base44's injected app params (`appId`/`appBaseUrl`) → SDK hits its default host → 404 on `me`/`batch`/all functions for fresh visitors (this caused the staged-login outage; fixed `0590`→`0591`). The committed `.env.production` bakes in `VITE_BASE44_APP_ID`/`VITE_BASE44_APP_BASE_URL`/`VITE_BASE44_BACKEND_URL` (public ids). **Verify after build:** `grep -o 68c22e8ff3726c063c4a53e2 dist/assets/index-*.js` must match. (A live **401 on `me` is NORMAL** = anonymous auth check; a **404 on `me`** = params missing.)
- **Functions:** `base44 --app-id … functions deploy <names…>` — **NEVER `--force`**.
- **GOTCHA:** the CLI's entity validator rejects the repo's older `user_condition` RLS format (~28 files)
  → before any deploy `mv base44/entities ../_entities_HOLD`, deploy, then move it back (git-tracked).
  We do NOT deploy entities. (Follow-up: migrate the RLS format so full `base44 deploy` works.)
- Production deploys are blocked by Claude's classifier → **owner runs the deploy commands**; Claude runs
  read-only CLI (`whoami`/`functions list`/`logs`). **Verify** by curling `app.serenovahub.com/?cb=$RANDOM`
  → check the `assets/index-*.js` hash + grep the version (don't trust the on-screen label alone).
- Full detail: decisions log v2.149 + the `serenova-deploy-test-flow` auto-memory.

**🧪 TEST BEFORE LIVE (no Base44 staging-deploy exists via CLI):** (1) **`base44 dev`** = app LOCALLY vs the
live backend (best frontend gate); (2) **Base44 editor Preview** still works with real Lisa/staged data —
but pull git→editor first via the owner's **`syncFromGitHub` function**, and note Preview hits the LIVE
functions; (3) live + curl verify. For backend changes (no preview-fn env) keep them **additive +
try/catch-isolated** so a direct-to-live deploy can't break existing users.

**✅ DONE this session (`0.6.0585`→`0591`, all live via CLI):**
- **MobileHub Live Flight bar** — peek bar on Home → full live-status sheet (delay/gate/terminal/alerts);
  cache-first (`getUserUpcomingFlightsWithLiveData`, 15-min TTL) to bound AeroAPI cost/rate-limits.
- **Staged band/crew get it too** — `LiveFlightData` is per-`flight_id` (shared per-segment). `getStagedMobileData`
  attaches live status for the member's own imminent flights (additive/try-caught; seeds via AeroAPI when
  stale; own-bookings-only → no traveller-less flights); `fetchLiveFlightData` got an assigned-traveller
  guard; the bar reads the shared record for staged. **Verified live: mj.oriole20 sees the 11:30 banner.**
- **PnrSegmentManager flight lookup (`0589`)** — AeroAPI `/schedules` (`flightLookup`) in Add/Replace
  autofills route/times/tz (de-dups onto the shared `Flight` → shared live cache); airport-tz-correct
  `datetime-local`; "Check for schedule updates" button (schedule refresh ≠ day-of live). Use **Replace
  Segment + Look up flight** to move a traveller onto the shared flight (e.g. Lisa → tomorrow's 11:30).
- **Staged login fixes (`0589`→`0591`)** — (1) `sendCode` retries 3× past a cold-start 401; (2) **the big
  one: `.env.production` + origin fallback** fixed the CLI-build 404-on-all-`/api` outage (see deploy
  block above). **Login VERIFIED working** (mj.oriole20 → SMS code → in). Note `requestStagedCode` sends
  **SMS** when a phone is on file (mj ends `4603`); email is the fallback / manual switch.

**👉 NEXT:** (a) tomorrow's `mj.oriole20` 11:30 flight = 2nd live check once in the 24h window. (b) **Push
Group 3** = proactive delay/gate PUSH via the **AeroAPI Alerts webhook → Supabase Edge Function** receiver
(decided direction, decisions log v2.148 — NOT polling). (c) entity-RLS-format migration so full
`base44 deploy` works without the entities move-aside. (d) **Phase C — Calendar Integration** SCOPED &
ready to build (spec `docs/CALENDAR-INTEGRATION-WORKING-DOC.md`; build phases v2.65): read-only tokenized
one-way ICS feeds by role (Personal→Tour→Account→AMC "Busy — City"), schedule-only/never-financials,
portable off Base44 (no `Core.*`); floating timezones + AMC-busy-from-actual-events resolved both open
decisions → **C1 (Personal) has no prereqs, start there**. (e) **Staged-login cold-start at the source** —
add a service-role-read retry inside `requestStagedCode`/`verifyStagedCode` (the frontend 3× retry is a
band-aid; needs a function deploy). (f) **`fa_flight_id` as canonical id** in `fetchLiveFlightData`
(FlightAware-recommended; currently matches airline+number+route).

---

## 🌅 MORNING START — (2026-06-23 AM — SUPERSEDED by the TOP BRIEF above; kept for history)

> **You may be on DESKTOP now. Laptop pushed up to `0.6.0584` / commit `65e0e16` last night.**

**Do this before anything else:**
1. **`git pull` BOTH repos** — the code repo (`serenovahub_b44`) AND this memory repo
   (`perplexity_project_memory`). The laptop is ahead; pull to avoid divergence. (`git pull --rebase`.)
2. **Confirm you're at `0.6.0584`** — `grep serenova src/version.js` should show `0.6.0584`.
3. Build sanity (optional): `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build` (green as of last night).
   **Deploy `0.6.0584`** (not earlier). Note: the event-detail Day Schedule synthesis (`0.6.0582`/`83`)
   was already verified live last night and looked great; the FINAL `0.6.0584` (home Today's-Schedule
   parity + hotel day-filter + Upcoming-Travel reorder) is the only un-deployed piece.

**State of play:** A long laptop session (2026-06-22 PM → 01:10 EDT) did a big **MobileHub travel
personalization + itinerary polish** arc (`0.6.0577`→`0.6.0584`), ALL pushed + build-green. The
event-detail half was verified live last night; the FINAL `0.6.0584` (home parity + hotel day-filter) is
**⏳ pending ONE live deploy + verify** (editor add-space-Save → rebuild → canary `v0.6.0584`). Everything
is **frontend-only** — no new Base44 functions/entities/fields, so a plain rebuild is all it needs.

**🧪 FIRST THING TO DO after deploying = live-verify the whole arc on a real device** (PWA on
`app.serenovahub.com`; test as a band member e.g. `mj.oriole20@gmail.com` AND as the artist owner Lisa):
- **Home:** "Upcoming Travel" card under Current/Next Event shows YOUR next trips; week-strip/today show
  only your travel.
- **Travel tab:** card-per-day, time-first rows, plane/car/train/bus icons, conf/PNR tag, **times in the
  correct (airport-local) zone**; band/crew see only their own flights/hotels (strict).
- **Hotel tab:** only your own hotel(s).
- **Drawers:** "My Reservation" (+ co-passengers sharing your booking ref) + "Other Passengers"/"Others
  Also Here" names; **ground drawer shows names not emails** (event-detail path).
- **Event detail → Travel card:** flights+ground merged, time leads, YOUR own pinned with a **teal "You"**
  tag; band/crew collapse others, Artist/admin see all under "Others".
- **Event detail → Day Schedule:** YOUR own hotel check-out/in + flight check-in/depart/arrive + ground
  now appear inline, time-sorted with icons (works for Artist too). Non-travelling admin = manual items only.
- **Home → Today's Schedule (`0.6.0584`):** now IDENTICAL to event-detail (same `buildDaySchedule` util) —
  time-first, icons, synthesized flight/ground/hotel; **home times now correct** (were wrong before).
- **Home order (`0.6.0584`):** Upcoming Travel sits **below** Today's Schedule.
- **Event detail → Hotel card (`0.6.0584`):** shows only the hotel(s) ACTIVE that day (stay covers the
  selected date) — a future stay no longer shows on earlier days.

**Known minor (logged, not blocking):** the **Travel-TAB** detail drawer still shows the raw email if a
traveller's `name_on_ticket` is blank (the tab has no single-event roster to resolve against — the
event-detail drawer DOES resolve). Easy follow-up: load member profiles at MobileHub level + pass a
resolver down. The owner is aware.

**⭐ TIME-SENSITIVE next build (still unclaimed): Live flight updates in MobileHub** — Lisa flies
**2026-06-23 (TODAY)** and has the PWA. See `LAPTOP-SESSION-NEXT.md` Item 2: reuse web
`LiveFlightTracker.jsx` + `getUserUpcomingFlightsWithLiveData`/`fetchLiveFlightData` (`AEROAPI_KEY_LIVE`,
day-of window) → live status block in `MobileTravelDetailDrawer`. On-demand-on-open for the demo.

**Standing rules:** `git pull --rebase` before each push; build with `./node_modules/.bin/vite build`;
bump `src/version.js` every deploy; commit author **Adam Jones <aj@adamdcjones.com>** ONLY (NO Claude co-author
trailer — owner owns authorship); confirm the file list before any push; Base44 **"Synced" ≠ deployed** → editor add-space-Save.

---

## 🟢 CURRENT STATE — 2026-06-22 PM (read this first; supersedes everything below)

> **Serenova `0.6.0584`** · latest `main` commit **`65e0e16`** · build green via
> `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`. Docs: decisions log **v2.147**,
> EventDetails working doc (**redesign COMPLETE**), `SERENOVA-MACHINE-SYNC.md`, and **`LAPTOP-SESSION-NEXT.md`**.

**✅ DONE 2026-06-22 — MobileHub personal travel/hotel scope for band/crew (`0.6.0577`→`0.6.0578`, pushed).**
Roster-scoping narrowed band/crew to their events; this narrows one level further — to the individual.
`src/utils/personalTravelScope.js` (match: flights `EventFlightBooking.user_email`, ground
`TravelSegment.travelers[].user_email`, hotels `Accommodation.rooming_list[].occupants[].email` + exact
name fallback). Applies to `band_member`/`crew_member`/`roadie_member` only; mgmt/TM keep full view; events
stay rostered.
- **Travel tab, Hotel tab, home week-strip/day-schedule, "Tonight" hotel** → own items only, **STRICT**
  (round-2 reversed the earlier per-event hotel fallback per owner live test — only what you're
  assigned to).
- **Detail drawers:** "My Reservation" (flights group anyone sharing your booking ref) + "Other
  Passengers"/"Others Also Here" name lists (`splitFlightBookings`/`splitRooms`). Travel tab passes the
  FULL `flightBookings` to the drawer (list stays strict) so co-passengers resolve.
- **Event detail shows ALL** but pins + **solid-teal** "You"-tags the member's own and collapses others
  behind "Show N other …"; if on NOTHING in a section → a "{who} is flying · Show flights" summary
  (`CollapsedSummary`).
Frontend-only — plain rebuild. ⏳ Live-verify: staged/logged-in band member sees only their own on
tabs/home; drawers show My Reservation + others; event-detail pins own / summarizes when not assigned.
- **Round 3 (`0.6.0579`):** (a) ground-transport traveller label was showing the raw email prefix when
  `name_on_ticket` was blank → `nameForEmail()` resolves email→display name via roster `memberProfiles` +
  `getDisplayName` ("Lisa F"). (b) New compact **"Upcoming Travel"** card on Home under Current/Next Event
  (next 3 trips after today, "6/24 ✈ Flight: JFK → SEA", own-only, taps to Travel tab).
- **Round 4 (`0.6.0580`):** (a) **"You" pin is now UNIVERSAL** — Artist/admin get it too. New
  `splitItems()` mode `pinnedOpen` (admin pins own highlighted + sees all others under "Others"); band/
  crew still `collapse`/`summary`. Home glances use new `mine` (viewer's own, everyone); tabs use
  `personal` (band own / admin full). (b) **Travel-tab redesign** (`MobileTravelList`): card-per-day,
  time-first compact rows, type icons (plane/car/train/bus), conf/PNR tag, ascending; **tz fix** — times
  now `formatAirportTime` (airport-local), not browser-local. (c) travel detail-drawer resolves traveller
  email→name via roster (`nameForEmail` from `MobileEventDetail`). **Known minor (still open):** the
  Travel-TAB drawer shows raw email if `name_on_ticket` blank (no single-event roster there).
- **Round 5 (`0.6.0581`):** event-detail itinerary cleanup — **Flights + Ground merged into one "Travel"
  card** (`dayTravelItems`, icon = type plane/car/train/bus), rows **lead with departure time** + arrival
  hint, pin/expand preserved; times via `formatAirportTime`. **Day Schedule:** time leads, transport rows
  get a car icon (`type==="transport"`), duration right-aligned when `end_time` exists. Hotel untouched.
- **Round 6 (`0.6.0582`→`0.6.0583`):** **Day Schedule now SYNTHESIZES the viewer's own travel+hotel** into
  the timeline (`dayScheduleAll`) — hotel check-out/in (Building2 purple), flight check-in (Clock orange,
  120/180min lead via `route_type`)/depart (Plane)/arrive (PlaneLanding), ground (car) — web
  `EventSchedule.jsx` parity; airport-local times via `localParts`+`formatAirportTime`. **Own-scoped by
  email → runs for EVERYONE so the performing Artist + band/crew see THEIR trips, non-travelling admin
  gets nothing** (owner reminder: treat Artist like band/crew). `0.6.0583` = TDZ hotfix (memo order).
- **Round 7 (`0.6.0584`):** extracted **`src/utils/itinerarySchedule.js`** (`buildDaySchedule`+
  `SCHEDULE_ICONS`) shared by event-detail + **home Today's Schedule** (now identical; fixes home's wrong
  times). **Upcoming Travel** moved below Today's Schedule. **Event-detail Hotel section day-filtered**
  (`dayAccomm` = stay covers selected day). Tonight card untouched.

**✅ DONE 2026-06-22 (`0.6.0576`, `7a93173`, pushed) — Item 1: MobileHub home nudge.** New
`src/components/mobile/MobileHomeNudge.jsx` at the top of the Home tab (above guest notices/week strip):
ONE dismissible banner — **Add to Home Screen** if not a PWA (native prompt Android/Chrome; instruction
sheet iOS — Apple exposes no programmatic install) **or Enable notifications** if standalone-without-push
(`enablePush()`, reuses Push Group 1 + `public/sw.js`). Snooze ~3 days per type via localStorage
(`nudge_install_until`/`nudge_push_until`); push only nudged while permission is `'default'`. **Frontend-
only — plain rebuild, no fn/entity deploy.** Live-only on `app.serenovahub.com` (renders nothing in the
preview iframe). ⏳ **Live-verify pending:** deploy (editor add-space-Save → canary `v0.6.0576`), then on
an installed PWA confirm the notifications nudge appears + enables; on iOS Safari (non-installed) confirm
the Add-to-Home-Screen sheet; dismiss → gone for 3 days.

**👉 NEXT (owner priority) — see `LAPTOP-SESSION-NEXT.md`:**
1. ~~**MobileHub home nudge**~~ ✅ DONE (`0.6.0576`, above).
2. **Live flight updates in MobileHub** — delay/gate/status on the mobile flight drawer; **Lisa flies
   2026-06-23 and has the PWA**, so it's demoable. Reuse web `LiveFlightTracker.jsx` +
   `getUserUpcomingFlightsWithLiveData`/`fetchLiveFlightData` (`AEROAPI_KEY_LIVE`, day-of window) →
   status block in `MobileTravelDetailDrawer`. On-demand-on-open for the demo; precursor to Push Group 3.

**Done this desktop session (2026-06-22, `0.6.0570`→`0.6.0575`, all pushed + mostly verified live):**
- **EventDetails redesign — COMPLETE.** #10 **Set List popup** (`0.6.0570`); #10 follow-ups
  (`0.6.0571`): **Contacts popup overflow fix**, set-list **default name** "{venue} - {Month Year}",
  **delete set list**, + **`role_templates.perm_setlist`='full' for artist admin/TM** (live Supabase —
  was `none`, so admin/TM couldn't add; band_member stays `edit`). **#7c** (`0.6.0572`/`0573`):
  all-hotels driving list + **hotel→airport leg** (`getVenueRouteInfo` + new `Accommodation.route_to_airport`
  field) — **deployed + VERIFIED LIVE**.
- **MobileHub improvements:** **venue route + drive/airport times** on the mobile event-detail Venue tab
  ("Getting There" block, reuses the deployed `getVenueRouteInfo`; works staged + logged-in) (`0.6.0574`);
  **venue-contacts source fix** (`0.6.0575`) — mobile read the legacy `venue_snapshot.day_of_contacts`,
  now reads `venue.contacts[]` (the redesign's source) + handles multi-role `roles[]`.
- **Authorship:** added proprietary **`LICENSE`** + **`SIGNATURE.md`** (overt RUSHMERE/KABINGA markers —
  display-only, NEVER in security/randomness code) + a CLAUDE.md "Authorship" pointer. No markers in code yet.
- **B4.2 (Verify-gated phone/email change) — ON HOLD** (owner paused on the email re-key risk); full
  plan preserved in the decisions log + machine-sync.

**Deploy note:** everything above is live-pushable with a **plain rebuild** (canary `v0.6.0575`) — the
only backend deploys this session (#7c Part B fn + field, setlist perm) are already done/applied.

---

## 🟢 CURRENT STATE — 2026-06-22 (historical — superseded by the PM brief above)

> **Serenova `0.6.0569`** · latest `main` commit **`84d5a59`** · build green via
> `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`. Docs in-repo: decisions log
> **v2.135**, `EVENTDETAILS-REDESIGN-WORKING-DOC.md` (source of truth for the redesign),
> `PUSH-TRAVEL-ALERTS-WORKING-DOC.md`, `STAGED-ACCESS-WORKING-DOC.md`, `MOBILEHUB-WORKING-DOC.md`.
> **Local memory clone is now `/Users/adamjones/Developer/perplexity_project_memory`** (the
> `/Volumes/adamjones/...` external mount is gone — use the `/Users` path).
>
> **🎯 SESSION (laptop, 2026-06-21 PM → 2026-06-22): EventDetails (web) redesign — BIG STRETCH DONE,
> owner on break; task RELEASED.** Source of truth = `docs/EVENTDETAILS-REDESIGN-WORKING-DOC.md` +
> decisions log **v2.134**. Page-scoped brand-teal restyle (rest of app rebrands later). Shipped
> `0.6.0541`→**`0.6.0568`**, all on `main`, build green:
> - **Tabs + header** — 4-tab scaffold (Overview · People · Travel · **Data Room**) + brand-teal
>   "Option-3" header (Back in header, quiet status, date range across performances).
> - **Venue card + #7 route map** — venue-focused card; map draws the venue↔hotel **route + 🚶walk/
>   🚗drive time + distance** via `getVenueRouteInfo` (Distance Matrix, cached on
>   `Accommodation.route_to_venue`); switched to a **coordinate** route (geocoded) so it never world-
>   zooms, **falls back to venue pin**. **Primary hotel** auto-picked (artist→longest stay→most
>   occupants) with a **manual dropdown** override (`Event.primary_accommodation_id`, optimistic, no
>   reload). **Key Contacts** (Advance/Tech) under the map.
> - **Layout** — **TM quick-access** in the tab bar (multi-TM + ⭐ make-lead); **Management→People**;
>   **itinerary full-width** (+ density tighten, dedupe duplicate hotel check-in/out rows, drop wrong
>   ground-transport tz); **reference cards moved ABOVE the itinerary + colorized** (Contacts blue, Set
>   List purple, Guest List teal).
> - **Guest list** — card → **popup** (open to EVERYONE to view; admin/artist/TM edit/approve/deny;
>   color ✓/✗/+ buttons; **Export CSV** dropdown; inline add/comps).
> - **Contacts** — **All Contacts popup** (Tour Manager · Team · Venue · Management, colorized, initials
>   avatars, tap-to-call/email).
> - **#6 Full Venue Info** — rebuilt: **tabbed** (Contacts/Parking/Tech Specs/Notes), reads the rich
>   **Venue entity** (was sparse `venue_snapshot`). **Venue Contacts** fully redesigned: show ALL venue
>   contacts + per-row **on-event toggle**, **inline single-row edit**, **multi-role tags** + Other,
>   **phone extension** (shown "x233") + **country-aware phone formatting** (libphonenumber).
>
> **✅ BASE44 DEPLOYS — ALL 4 DONE + verified live 2026-06-22:** `getVenueRouteInfo` redeployed
> (coordinate route works); `Accommodation` (`route_to_venue`+coords), `Event` (`primary_accommodation_id`),
> `Venue` (`contacts[].roles[]`+`contacts[].phone_ext`) synced. Coordinate route, primary-hotel dropdown
> persistence, and venue-contact role-tags/extension all confirmed working live. (`GOOGLE_PLACES_API_KEY`
> in secrets, unrestricted; Geocoding API enabled.)
> **✅ #9 + Stage 4 DONE (`0.6.0569`):** Travel-tab accommodations **grouped by hotel**
> (`src/utils/accommodationGroups.js`) — one listing per hotel + a row per stay (date-only +occupants);
> #9 Overview summary already covered by the venue card. **REMAINING redesign (unclaimed):** #10 Set List
> popup (+ tidy unused `getStatusBadge`/`getContractBadge`), #7c all-hotels/airport map mode.
>
> **✅ CLOSED 2026-06-21 (owner-confirmed working): Guest notifications + Tour Manager picker + mobile
> approve/deny** (`0.6.0527`→`0.6.0533`). Full guest flow works end-to-end: request → notify TM(s)/AMC
> → approve/deny on web **and** mobile → requester notified (email+push+banner). Push Group 1 verified
> live; Group 2 + TM picker + mobile approve/deny owner-confirmed (mobile approve/deny `0.6.0533` to
> re-verify on next deploy). Detail in the dated blocks below.
>
> **✅ Travel ground-transport fix DONE 2026-06-21 (owner-confirmed "works and saves," `0.6.0535`):**
> two root causes — (1) link-only mgmt user (`adam@originalartists.com` via Original Artists) 403'd
> `getUsersByEmailsForAccount` → fatal red-error page (broadened that fn's access + made Travel
> enrichment non-fatal; also clears the TM-picker/Team enrichment caveat for link-only mgmt users);
> (2) `AddTravel` mis-wired `TravelForm` (React `ref` to a non-forwardRef fn → `?.save()` silent no-op)
> → fixed to `formRef`/`accountId`/`onSave`/`requestSubmit()`.
>
> **🗂️ EventDetails redesign — DETAILED STAGE LOG (historical; SUPERSEDED by the SESSION summary at the
> top of CURRENT STATE — as of session end the redesign is RELEASED at `0.6.0568`). Kept for per-stage
> detail.** Source of truth = `docs/EVENTDETAILS-REDESIGN-WORKING-DOC.md`. Page-scoped brand-teal restyle
> (the rest of the app rebrands later "as we update"). Done so far: **Stage 1** 4-tab scaffold
> (Overview · People · Travel · Financials, `0.6.0541`) + polish (slim teal header, Quick Actions above
> tabs/gated/collapsible, `0.6.0542`); **header "Option 3"** (`0.6.0544`, owner-picked — Back in header,
> action buttons top-right, smaller iconless title, date RANGE across performances, quiet status
> dot+label); **Stage 5 Venue card** (`0.6.0545`) — `VenueDetailsCard` rebuilt venue-focused: dropped
> the duplicated event title; venue name + Database badge + capacity, address, **small keyless Google
> Maps embed** (`maps?q=…&output=embed`, **no API key / no per-view cost**; owner has a Google Places
> key but keyless is free+fine — upgrade path is enabling **Maps Embed API** for the official
> `maps/embed/v1/place` endpoint if it ever flakes), Open-in-Maps, **venue contacts / parking / house
> backline** from the `Venue` entity; the **hotel moved OUT to the Travel tab** (`AccommodationCard`
> under `TravelCard`, interim flat list); **#7 venue↔hotel ROUTE map** — **7a** (`0.6.0546`) keyless
> route on the venue card + primary-hotel pick (artist's hotel → longest stay → most occupants,
> `src/utils/primaryAccommodation.js`); **7b** (`0.6.0547`, code-done, ⏳ needs Base44 deploy — see
> OUTSTANDING above) `getVenueRouteInfo` fn (Distance Matrix walk+drive, caches `route_to_venue`) +
> 🚶/🚗/distance readout. **STILL OPEN (next, owner-prioritized stages 6–14 in the redesign doc):**
> the bulk of the Overview-v2 brainstorm — #6 Full Venue Info dialog cleanup, #8 Advance/Tech quick
> contacts, #9 accommodation summary on Overview (decided Overview+Travel), #10 Set/Guest popup buttons,
> #11 Management/Representation → People, #12 **persistent Tour Manager quick-access** in the tab bar,
> #13 itinerary full-width, #14 **Financials → "Data Room"** + Linked Files inside; **#7c** (all hotels
> on map / >1-hotel driving-only + hotel→airport) deferred. Plus Stage 2/3 (`GuestManagerDialog`),
> Stage 4 (hotel-grouped accommodations on Travel), and cleanup of unused
> `getStatusBadge`/`getContractBadge` in `EventDetails.jsx`. **Expenses still parked. Push Group 3
> (flight alerts) = later.** See `SERENOVA-MACHINE-SYNC.md` Active-work table.

**Done & verified live this stretch (all pushed to `main`):**
1. **Phase SA C part 1 — band-member staged GUEST REQUESTS ✅ VERIFIED LIVE** (`0.6.0519`→`0521`).
   Staged band member requests a guest from MobileHub → admin Approves/Denies on the EventDetails
   **`GuestListCard`** (new) or the EditEvent **`GuestListTab`** → requester notified on **3 channels**
   (email via `Core.SendEmail` + dismissible MobileHub **home banner** + web surfaces). Guests live in
   `event.guest_lists` as `Requested` (no comp until approved); added a **`Denied`** status; `usedSlots`
   excludes Requested+Denied. Owner/admin/manager on mobile get **both Add + Request**; band = Request
   only; staged = view-only. Service fns: `submitStagedGuestRequest`, `decideGuestRequest`,
   `dismissStagedNotice`.
2. **🔒 MobileHub roster-scoping ✅ VERIFIED LIVE** (`0.6.0523`). Band/crew were seeing the artist's
   ENTIRE calendar + all travel/hotels; now scoped to events where their email is in `roster_members`
   (+ tied travel/hotels + own travel bookings). `getStagedMobileData` is the **hard boundary** (staged
   band/crew have no API access); logged-in path scoped via a `scopeRef`. Roles scoped:
   `band_member`/`crew_member`/`roadie_member`. **Caveat:** roster accuracy is now load-bearing.
3. **Phase SA B4.1 — staged "My Profile" 🟡 SHIPPED, live-verify in progress** (`0.6.0524`→`0525`).
   Reached from the MobileHub hamburger (staged only). Tabs **About You / Address / Travel**. Edits
   LOW fields (name, known-as, mobile, office phone, **instrument + tour-role dropdowns** w/ "Other",
   DOB, home address) + **travel docs shown IN FULL** to the member's own code-verified session
   (passport/KTN/redress — owner decision 2026-06-20; **reversed the earlier masked plan**). **NO
   payment/bank/SSN path at all** (hard-excluded by a server whitelist — that's the security gate).
   `saveStagedProfile` rewritten (whitelist + `deepMerge` travel_info, preserves visas/loyalty);
   `getStagedMobileData.profile` returns the broader fields. `Membership.pending_profile_data` gained
   `known_as`+`date_of_birth` (additive). Polished from live test: safe-area top padding (menu X +
   back buttons were under the iOS status bar), date-input width fix, greeting refresh on save.

**⚠️ Pending LIVE verify (needs a Base44 rebuild — editor add-space-Save trick → hard reload; canary
`v0.6.0525`):** B4.1 — open My Profile as a staged band member (`mj.oriole20@gmail.com`), edit Low
fields + dropdowns + full travel docs → Save → reopen, all persisted; sensitive fields absent; menu/back
tap targets clear the notch. The **3 guest fns + `saveStagedProfile`/`getStagedMobileData` changes must
provision** on Base44 (watch the non-provisioning gotcha).

**🔜 NEXT (owner-queued, in order):**
- **B4.2 — Verify-gated phone + email change** (the natural next step; owner flagged that changing the
  SMS-login phone must require a code). Email change = code-to-new-email → backend re-keys all the
  identity's Membership records old→new. Phone change = Verify code to the new number (SMS is live, C2).
- **Phase SA C part 2 — EXPENSES** via a separate **`StagedRequests` holding queue** (owner's design:
  nothing financial enters an account until approved; service-role receipt quarantine + promote-on-
  approval — keeps the staged hard-cap fully closed on financials).
- **Push notifications + Travel/Transportation alerts** track. **Reconnaissance done:** the flight infra
  is ~80% there — **FlightAware AeroAPI** (two cost-tiered keys), a **`LiveFlightData` entity with a
  full alert model** (delay/gate/terminal/cancellation, severity, acknowledge flow),
  `refreshLiveDataForUpcomingFlights`, a `route_type` domestic/intl field, the 2h/3h arrival logic, and
  an extensible `UserPreference` entity. **Gaps:** the push delivery pipeline (service worker + VAPID +
  PushSubscription entity + opt-in), a **scheduler/cron**, and the **daily-summary generator**. Owner
  wants: flight alerts = **delays + gate changes only**; a **morning-of/prior-day travel summary** with
  check-in rec (2h domestic / 3h intl). ✅ **PLANNING DOC NOW WRITTEN** (laptop, 2026-06-21):
  `docs/PUSH-TRAVEL-ALERTS-WORKING-DOC.md` — 5 build groups + audit-grounded foundation. Owner
  decisions locked: alerts = delays+gate only; **day-of** summary; **on-entry live lookup** at
  flight-record time; **scheduler = GitHub Actions cron** (confirmed no native Base44 scheduler);
  tiered refresh cadence (T‑24h→T‑6h ~2h, T‑6h→landing ~15m). Decisions-log v2.98, build-phases v2.59.
  **Group 1 (push delivery) ✅ BUILT 2026-06-21** (Serenova `0.6.0526`, build-green, ⚠️ live-verify
  pending): new `PushSubscription` entity + `savePushSubscription`/`sendPushToSubscriptions`
  (`npm:web-push`+VAPID) + push-only `public/sw.js` + `src/lib/push.js` + "Enable notifications" in
  MobileHamburgerMenu (logged-in only; staged=v1.1). Owner added the 3 VAPID Base44 secrets; **still
  needs `VITE_VAPID_PUBLIC_KEY` in client env (.env.local)** + deploy (save-trick; new entity+2 fns
  must provision) + iPhone test. Runtime risk: `npm:web-push` under Deno → swap to Deno-native if it
  throws. **`0.6.0527` brought STAGED band/crew push forward into v1** (live test: the MobileHub menu
  is a staged session — the primary audience; `savePushSubscription`/`sendPushToSubscriptions` resolve
  identity from the staged session token, client passes `getStagedToken()`, menu gate relaxed to all).
  VAPID env fully wired (`VITE_VAPID_PUBLIC_KEY` committed). **✅ VERIFIED LIVE 2026-06-21** — a staged
  band/crew member on an installed PWA enabled notifications + received the self-test push end-to-end;
  `npm:web-push` runs fine under Base44's Deno (runtime risk cleared). **Push Group 1 DONE (`0.6.0527`).**
  **Push Group 2 🔄 BUILT (`0.6.0528`, verify pending):** `decideGuestRequest` now pushes the requester
  on approve/deny (inline web-push, reuses Group 1 VAPID + `PushSubscription`, best-effort) + records a
  new `Notification` entity row → the guest notice now goes out on **3 channels** (email + banner +
  push). **No new secret** (VAPID reused). ⚠️ deploy → new `Notification` entity + updated
  `decideGuestRequest` must provision → test as: staged band member requests guest → admin approves →
  requester gets a push. **`dispatchNotification` multi-channel dispatcher + prefs panel intentionally
  deferred to Group 3** (a send-to-anyone dispatcher needs the same secret-gated entry the cron does).
  **Also (`0.6.0529`): approver-notify on REQUEST** — `submitStagedGuestRequest` now notifies the
  approver(s) when a guest is requested (was silent): **event-assigned Tour Manager only, fallback to
  artist/owner + AMC confirm-role members** (owner rule); email + push + `Notification` row; requester
  never self-notified. Reuses Group 1 VAPID (no new secret). Follow-up: the logged-in (non-staged)
  request path is a client `Event.update` → no server notify (staged is the dominant band/crew path).
  **Next = Group 3** (GH-Actions cron + flight delay/gate alerts + on-entry lookup; builds the
  dispatcher with `NOTIFY_INTERNAL_SECRET`, consolidating the 3 inline web-push sites).
  **🧭 ALSO (`0.6.0530`): Tour Manager picker rebuilt + multi-TM.** Pre-existing bug (NOT push work,
  from `922785f` 2026-06-16): TM dropdowns listed direct artist-account members only → linked
  management-company people (Adam @ Original Artists) couldn't be selected as TM (hence the missing
  notification). Fix: new `src/utils/eventTeamCandidates.js` (artist team + all active linked AMC/BMC/
  booking members, deduped, tagged onEvent/relevant) + new searchable/grouped **multi-select**
  `src/components/events/TourManagerPicker.jsx` (Command+Popover; quick list "On this event" →
  "Tour & production"; search = everyone). Additive `Event.tour_manager_user_emails[]` (canonical;
  singular `tour_manager_user_email` mirrors `emails[0]` for back-compat; all readers use
  array-with-singular-fallback). Wired into EventWizard/BasicInfoTab/ManagementAgentsCard;
  `submitStagedGuestRequest` notifies ALL event TMs. **Post-test fixes (`0.6.0531`):** linked people
  now show their **artist-functional role** ("Tour Manager", from `LinkedAccount.artist_account_permission_template_id`
  via `Team.jsx` PRETTY_ROLE) not their company membership role (was "Owner"/"Admin"); candidate
  **names enriched from the User entity** (`getUsersByEmailsForAccount`, known_as > first+last) → "Adam
  Jones (TM)" not "adam". ⚠️ deploy (new entity field + fns provision) → retest: dropdown lists Adam
  (Tour Manager, full name), multi-select works, request notifies all TMs.
  **📱 Mobile guest approve/deny (`0.6.0533`, build-green):** logged-in TM/Admin on MobileHub can now
  **Approve/Deny** `Requested` guests inline in `MobileEventDetail` (gated `canManageGuests` =
  `!staged && canEdit('guest_list')`; via `decideGuestRequest`) — makes the guest push actionable on
  the phone. **NEXT (owner-queued, NOT expenses): EventDetails (web) redesign** — cleaner view/flow +
  INLINE guest management (add/edit/comps/approve-deny) so you don't open EditEvent. Recon done
  (`GuestListCard` = approve/deny-only; `GuestListTab` = full manager but coupled to EditEvent state →
  extract). Plan to follow.

**Decisions/scope locked recently:** web band/crew full-view = **DEFERRED** (band stay MobileHub-only;
not granting web access yet — keep the scoping pattern ready). Travel docs in staged = **full view**
(verified-session, owner-accepted). Email approval carries a SendGrid auto-unsubscribe footer (watch-item).

---

## 🟢 CURRENT STATE — 2026-06-19 PM (historical — superseded by the 2026-06-21 brief above)

> **Phase SA C part 1 — band-member staged GUEST REQUESTS ✅ SHIPPED & VERIFIED LIVE.** Serenova now
> **0.6.0523**. Latest `main` commit `8c93d8f`; build green via
> `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`. Docs: decisions log **v2.97**,
> build phases **v2.58**, plus `STAGED-ACCESS-WORKING-DOC.md` + `MOBILEHUB-WORKING-DOC.md`.

> **🔒 ALSO FIXED THIS SESSION (0.6.0523, verified live) — MobileHub roster-scoping.** Band/crew
> (`band_member`/`crew_member`/`roadie_member`) were seeing the artist's ENTIRE event calendar + all
> travel + all hotels. Now scoped to events where their email is in `roster_members` (+ tied
> travel/hotels + their own travel bookings); owner/admin/manager/tour_manager/finance still see all.
> **`getStagedMobileData` is the hard boundary** (staged band/crew have no API access); logged-in path
> scoped at the source via a `scopeRef`. **Caveats:** roster accuracy is now load-bearing (no roster
> entry → that show is hidden); logged-in scoping is UX-level only (Event RLS still allows API reads →
> hard lock is **Phase 0/F**); the **web Events/Dashboard has the same gap** — offered, owner to decide.

**What shipped this session (guest requests, the first half of Phase SA C):**
- **Flow:** a staged band member requests a guest from MobileHub → it lands in `event.guest_lists` as
  `status:'Requested'` (does NOT consume a comp until approved) → an admin/owner/manager **Approves or
  Denies** on the **EventDetails Guest List card** (new `GuestListCard`) or in the **EditEvent guest
  tab** (`GuestListTab`) → the requester is notified on **3 channels**: **email** (`Core.SendEmail`),
  a dismissible **MobileHub home banner**, and the web review surfaces. **VERIFIED LIVE end-to-end**
  (`mj.oriole20@gmail.com` band member: email received + banner shown + Confirmed + comp count moved).
- **Mobile Guests section is now role-aware:** logged-in **owner/admin/manager → BOTH "Add guest"**
  (direct, Pending, client `Event.update`) **and "Request"** (Requested, needs approval); **band
  members → "Request a guest"** only (staged via `submitStagedGuestRequest` service fn; logged-in via
  client write); **staged sessions stay view-only** (hard-cap). Per-show-time picker when multi-show.
- **New service fns:** `submitStagedGuestRequest` (token→band_member gate), `decideGuestRequest`
  (admin/owner/mgmt-seat auth → Confirmed/Denied + stamp + email), `dismissStagedNotice`. ⚠️ **These 3
  must provision on Base44** (watch the same non-provisioning gotcha `StagedSession` hit).
- **Model:** added a **`Denied`** status; `usedSlots` excludes Requested AND Denied (neither consumes a
  comp). `Event.jsonc` brought honest (additive: `performance_time`, status enum, request/decision
  fields). **No new entity, no migration** (`guest_lists` is JSON on `Event`).
- **Gating model:** REQUEST = account membership role `band_member`; ADD = logged-in
  `canEdit('guest_list')`; staged = view-only. The event-roster **band/crew label** (Add Personnel
  role-type) is per-event and independent of both account role AND guest gating (so an admin showing as
  "crew" on a roster is expected, not a bug).

**🔜 NEXT (owner-queued, in order):**
1. **PWA Web Push** track — real lock-screen notifications (service worker + VAPID + `PushSubscription`
   entity + opt-in UX + send fn); the decide fn then gains a 3rd channel. iOS needs the PWA installed
   to Home Screen + permission grant; live-only on `app.serenovahub.com`.
2. **Phase SA C part 2 — EXPENSES** via a **separate `StagedRequests` holding queue** (owner's design:
   nothing financial enters an account until approved; service-role receipt quarantine + promote-on-
   approval — keeps the staged hard-cap fully closed on financials).

**Watch-items (logged):** approval email carries a SendGrid/Base44 auto **unsubscribe** footer (odd on
a transactional notice — revisit when push lands); logged-in (non-staged) band-member requests use a
client write (fine); the optional "roster auto-suggests band/crew from account role" idea (out of scope).

**Reminders unchanged:** Base44 **"Synced" ≠ deployed** → editor add-space-Save trick → canary
`v0.6.0522`; `git pull --rebase` before each push; build with `./node_modules/.bin/vite build`; bump
`src/version.js` every deploy; commit author **Adam Jones <aj@adamdcjones.com>** ONLY (NO Claude co-author
trailer — owner owns authorship); confirm the file list before any push.

---

## 🟢 CURRENT STATE — 2026-06-19 (clean morning brief; superseded by the PM brief above)

> **2026-06-19 AM updates:** Login screens unified + rebranded (dark teal/charcoal bg + white card,
> square mark, "Serenova Hub" title, tan Band&Crew button, darker input/button borders). **ARCHITECTURE
> DECISION (decisions log v2.93):** keep `src/pages` flat (Base44 reserves/auto-discovers it — do NOT
> reorg into per-hub folders); components stay in domain subfolders; **per-hub branding via the new
> `src/config/hubThemes.js` registry** (pairs with `version.js`); migration lever = `src/api/*` seams.
> **Known Base44 limitation:** a cold hit to `/login` shows Base44's hosted auth (reserved path) — leave
> it; expected to disappear at migration off Base44. Serenova now **0.6.0518**.


**Next focus (owner): continue MobileHub.** The natural next build is **Phase SA C — band-member
staged writes** (guest requests + expense requests submitted from MobileHub via the staged session),
which the now-live staged identity unblocks. See `docs/MOBILEHUB-WORKING-DOC.md` (deferred Group 2c/2d)
+ `docs/STAGED-ACCESS-WORKING-DOC.md` (step C).

**Where we are — all built & pushed to `main` (latest commit `8b166c2` code / `befe37a` handoff), build
green via `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`:**
- **Phase SA (Staged passwordless access) — COMPLETE & VERIFIED LIVE:** A0 (Twilio test) · B1 (login/
  session/auth-gate) · B2 + **B2.1** (staged MobileHub data incl. event-detail contacts/setlist/venue) ·
  **B3** (first-login confirm card: name + **E.164** mobile via `react-phone-number-input`) · **C2**
  (SMS-preferred login: SMS-if-on-file + email fallback, masked hint) · **M** (SystemHub live Twilio
  cost card). Session token lives on `ShareableLink` (`link_type:'staged_session'`). **Rolling 30-day
  idle TTL**, **revoke-on-logout**, **5-session LRU cap**, **device labels**.
- **Per-hub versioning system** (`src/version.js` — the deploy canary; bump the affected hub each
  deploy). Current: Serenova **0.6.0518** · System **1.0.0019** · Venue 0.5.0018 · Contract 0.4.0010 ·
  Financial 0.3.0007 · Storage 0.2.0003. DB audit log = `AppVersion` (+ `recordAppVersion`).
- **Mobile routing fix** (1h `preferDesktop` TTL, centralized in Layout) + **"Band & Crew Sign In"**
  button on Login → `/StagedLogin`.
- **SystemHub user/account drill-in:** `UserDetailPanel` (profile + accounts/roles + staged sessions w/
  Force-logout) via `getUserContext`; Users tab → **All Users** (anyone) + **System Users**
  (admin-only); master-detail layout; header search + Support user cards open detail.
- **SystemHub Overview redesign:** hero KPIs trimmed + **Staged Users** card (Active big / Total below;
  Total = `status:'staged'` memberships ∪ session-havers = ~18; Active = used within 12mo). Failed
  Validations + Permission Issues moved to Platform Health. **Analytics** = non-clickable "Soon" item in
  the top `HubSwitcher` (future ONE global, filterable hub). `User.was_staged` groundwork added.

**⚠️ Pending LIVE verify (needs a Base44 rebuild — editor-save trick; then hard-reload):** canary =
login footer `v0.6.0514` / System Hub `v1.0.0019`. Spot-check: staged login by SMS+email; confirm card
E.164; logout revokes session; SystemHub user drill-in + Staged Users count; Twilio cost card live.

**Backlog / deferred (logged):** Phase SA **B4** (staged My Profile + 🔒 data-inventory gate) · **D**
(band upsell→full account port; sets `was_staged`) · build the **Analytics** hub · Platform Health rows
→ clickable own-pages · email deliverability (codes→iCloud spam: SendGrid domain auth/DMARC/warmup) ·
custom device naming for sessions · adopt `PhoneNumberField` on the other ~8 phone inputs ·
`MobileShareDialog` staged client-reads + "Switch to Web View" no-op for staged (minor) · Phase 9
cleanup (stale entities/legacy imports). Full roadmap: `docs/SERENOVA-BUILD-PHASES.md`.

**Critical reminders:** Base44 **"Synced" ≠ deployed** → force via editor add-space-Save; `git pull
--rebase` before each push (Base44 2-way-syncs). Build with `./node_modules/.bin/vite build` (plain
`npx vite build` may grab the wrong vite). **Bump `src/version.js` (affected hub) every deploy.** Commit
author **Adam Jones <aj@adamdcjones.com>** ONLY — **no Claude co-author trailer** (owner owns authorship). **Confirm the file list
before any push.** Docs: decisions log **v2.90**, build phases, working docs.

---

## 🟡 2026-06-19: Staged Users metric fix + Analytics → global hub placeholder (build-green)
- **Staged count fix** (`systemHubStats`): Total = distinct staged-setup identities (Membership
  `status:'staged'`, 19 in DB) ∪ session-havers; Active = those with a session used within 12mo. Card
  now ~1 active / ~20 total (was wrongly "1"). **Card emphasis flipped:** big = Active Staged, total below.
- **`User.was_staged`** field added (groundwork) — set during Phase SA D port for staged→full-user
  conversion analytics. Inert until D.
- **Analytics = ONE global hub** (recommended, owner-aligned): non-clickable "Analytics · Soon" added to
  the top-level `HubSwitcher` (sibling to System Hub/Venues/Storage); header chip removed. Cross-hub,
  filterable — not per-hub.
- System Hub `1.0.0018`→**`1.0.0019`**. Build green.
- **Deferred (owner):** Platform Health rows → clickable own-pages (later phase); build the Analytics hub;
  Phase SA D port sets `was_staged`; full staged analytics (lifetime/active/converted) lives in Analytics.

## 🟡 2026-06-19: SystemHub Overview redesign (build-green, verify pending)
- Hero KPIs trimmed to headline numbers + new **Staged Users** card (Total + Active%; **Active = within
  12 months**). New `systemHubStats` fields `stagedUsersTotal`/`stagedUsersActive` (aggregated from
  staged_session links by email).
- **Permission Issues + Failed Validations moved out of the hero badges into Platform Health** (with
  detail counts) — `HubOverview.jsx`.
- **Analytics** = non-clickable "Analytics · Soon" chip in the SystemHub header top-right (placeholder
  for a future top-level hub; deferred/stubbed, not built).
- System Hub `1.0.0017`→**`1.0.0018`**. Build green.
- **Verify:** Staged Users card shows real total/active%; Failed Validations + Permission Issues now
  under Platform Health; Analytics chip non-interactive.

## 🟡 2026-06-19: Staged session hygiene (build-green, verify pending)
A tester had 9 active sessions: every login mints one + logout only cleared the local token (server
link stayed active). Fixed:
- **Revoke on logout** server-side: new `base44/functions/endStagedSession/entry.ts` + `endStagedSession()`
  in `stagedSession.js`; MobileHub logout calls it for staged users.
- **5-session LRU cap**: `verifyStagedCode` deactivates oldest active beyond the 5 most recent (keeps
  multi-device, bounds sprawl). Existing piles self-trim on next login.
- **Device labels**: `deviceLabel()` (UA → "iPhone · Safari") sent to `verifyStagedCode`, stored in new
  `ShareableLink.device_label` (+ `staged_session` added to the link_type enum). Browsers can't give
  exact model; custom names deferred.
- **Collapsed sessions UI** in `UserDetailPanel` ("N active — click to expand" → device list).
- Serenova `0.6.0513`→**`0.6.0514`**, System `1.0.0016`→**`1.0.0017`**. Build green.
- **NEXT (confirmed):** Overview — add **Staged Users** KPI card (Total + Active%; **Active = within 12
  months**) + move Failed Validations/Permission Issues into Platform Health. **Deferred:** separate
  top-level **Analytics** hub.

## 🟡 2026-06-19: SystemHub user/account drill-in (build-green, verify pending)
SystemHub could find but not OPEN users (header search/Support/Users all dead-ended; account detail in
Support already worked). Built:
- **`getUserContext`** fn (email → user + all memberships/accounts + staged sessions; system-role gated).
- **`UserDetailPanel`** (`src/components/systemhub/`) — profile + accounts/roles + staged sessions w/
  Force-logout (reuses `adminStagedSessions`). **`AllUsersAdmin`** — all-users search → detail.
- **Users tab → sub-tabs (owner call):** **All Users** (anyone) + **System Users** (`SystemUsersManager`,
  **system_admin only**, not support).
- **Wired:** header search → user routes to Users›All Users (preselected), account routes to Support
  (new `initialAccount` prop) — both open the detail; Support user cards now clickable → UserDetailPanel.
- System Hub `1.0.0014`→**`1.0.0015`**. Build green (`./node_modules/.bin/vite build`).
- **Deferred (owner idea):** support-flow "transfer support / add system user to a support issue".
- **⚠️ Verify:** search a band/crew user (e.g. `jones_adamd`) → detail shows accounts/roles + staged
  sessions + Force-logout; non-admin shouldn't see the System Users sub-tab; header-search clicks open detail.

## 🟡 2026-06-18: Staged session rolling TTL + admin session controls (build-green, verify pending)
- **Rolling 30-day idle TTL** (was fixed-30-from-login): TTL now measured vs `updated_date`;
  `validateStagedSession` touches the link on a valid check (throttled ≤1×/day) so the window rolls
  with use → active band/crew never re-verify, only 30-days-unused expires. All 3 TTL checks
  (`validateStagedSession`/`getStagedMobileData`/`saveStagedProfile`) use `updated_date||created_date`.
- **Admin controls:** new `base44/functions/adminStagedSessions/entry.ts` (system-admin: list / revoke
  one email / revoke_all) + "Staged login sessions" card in `CostUsageAdmin` (SystemHub → Cost & Usage):
  look up sessions, **Force log out**, **Revoke ALL** (confirm) for post-build forced re-verify.
- Serenova `0.6.0512`→**`0.6.0513`**, System `1.0.0013`→**`1.0.0014`**. Build green.
- **⚠️ Verify after deploy:** the rolling touch relies on Base44 auto-bumping `updated_date` on
  `.update()` — confirm a session's `updated_date` advances on app open (and stays valid past 30 days
  of active use). Admin: force-logout `jones_adamd` → next open should require re-verify.

## ✅ 2026-06-18: Phase SA M VERIFIED LIVE + double-count fix
SystemHub Twilio cost card live (owner added secrets; the only snag was a TWILLO→TWILIO typo on the
account-SID secret name). **Accuracy fix:** `getTwilioUsage.summarize()` now takes the total from
Twilio's authoritative **`totalprice`** record + builds a **leaf-only** breakdown (rollup denylist:
`totalprice`/`account-security`/`authy-phone-verifications`/`pv`); returns `{totalCents, leafSumCents,
categories}`. Card adds an "Other (tax/unitemized)" reconciliation row + Total row + estimate caveat.
Month now = **$0.85**, matches the Twilio console (was over-counted ~4× by flat-summing the tree).
System Hub `1.0.0012`→**`1.0.0013`**. **Phase SA roadmap A0/B1/B2/B2.1/B3/C2/M ALL ✅.** Remaining: B4
(My Profile + data-inventory gate), D (upsell/port).
- Build with `./node_modules/.bin/vite build` (plain `npx vite build` transiently grabbed vite@8 and
  failed on entry resolution — env hiccup; CI uses project vite).

---

## 🟡 2026-06-18: Phase SA M BUILT — SystemHub live Twilio cost card (secrets added → verify after deploy)
New `base44/functions/getTwilioUsage/entry.ts` (system-admin gated) pulls the **Twilio Usage Records
API** (Today/ThisMonth/AllTime, real billed `price` per category). New live card in
`src/components/admin/CostUsageAdmin.jsx` (SystemHub → Cost & Usage): spend tiles + this-month
category table + Refresh; graceful not_configured/forbidden/error states. System Hub `1.0.0009`→
**`1.0.0011`**. Build green.
- **Owner ADDED the secrets 2026-06-18:** `TWILIO_ACCOUNT_SID` + `TWILIO_USAGE_KEY_SID` +
  `TWILIO_USAGE_KEY_SECRET` (a Standard API key). Function also auto-discovers the account SID from
  the key (`/Accounts.json`) if the SID secret is absent — so the key pair alone is enough.
- **⚠️ Verify spelling** = exactly `TWILIO_…` (T-W-I-L-I-O), not "TWILLO". After deploy: SystemHub →
  Cost & Usage → Twilio card → Refresh should show live Today/Month/AllTime spend + categories.
- Build note: local `npx vite build` transiently grabbed vite@8.0.16 and failed (entry resolution);
  `./node_modules/.bin/vite build` is green — environment hiccup, not code. CI uses the project vite.

---

## ✅ 2026-06-18: Phase SA C2 (+ B3) VERIFIED LIVE — SMS-preferred staged login works
Owner "working great": `jones_adamd@me.com` (mobile `+13108904603`) gets the code by **SMS** with the
masked hint, the email-instead toggle works, both channels sign in. Fixes codes-→-iCloud-spam.
**Phase SA email/SMS code login fully live: A0, B1, B2, B2.1, B3, C2 all ✅.**
**Remaining Phase SA:** B4 (My Profile + data-inventory security gate), D (band-member upsell/port),
M (SystemHub Twilio cost card — needs a Twilio Usage-read credential; SMS-default raises spend).

---

## 🟡 2026-06-18: Phase SA C2 BUILT — SMS-preferred staged login (build-green, verify pending)
Owner: at login, send the code by SMS if a phone is on file (fixes iCloud email-spam), email fallback.
SMS unblocked (Verify 10DLC-exempt, A0 proved it, numbers now E.164).
- **`requestStagedCode`** channel `auto` (default) → SMS if valid E.164 mobile on membership, else
  email; returns `{channel, hint}` (masked last-4). SMS-fail auto-falls-back to email.
  Anti-enumeration: no-access == access-without-phone response (only "phone on file" distinguishable —
  owner accepted, masked-hint chosen).
- **`verifyStagedCode`** re-resolves verify `To` per channel server-side (client never sends phone).
- **`StagedLogin`**: "Code sent to your phone ending in ••••4603" + channel toggle (email/text);
  re-sends on switch; verify passes channel. `stagedSession.requestStagedCode` default → `auto`.
- Serenova `0.6.0511`→**`0.6.0512`**. Build green.
- **⚠️ Verify:** `jones_adamd` (now `+13108904603`) should get the code by **SMS** with masked hint;
  "email instead" toggle works; both channels verify + sign in. Watch SMS deliverability/cost (M card).
- **Next:** M (Twilio cost card — needs Usage-read credential), D (band-member upsell/port), B4 (My
  Profile + data-inventory gate). Phone field can be adopted by the other ~8 plain-text phone inputs.

---

## 🟡 2026-06-18: Phone field fixed dial-code prefix + reset-not-showing diagnosis
- `PhoneNumberField` now `countryCallingCodeEditable={false}` → fixed `+1`/`+44` prefix in the one
  box, change country via the flag (owner's "(+1 ____)" ask). UK trunk-0 dropping is automatic
  (libphonenumber-js). Serenova `0.6.0510`→**`0.6.0511`**, build green.
- **Confirm-card "didn't pop" for jones_adamd = NOT data:** DB flag verified `null` (reset held). A
  warm PWA keeps React state → no re-fetch; **cold reopen / logout-login** shows the card. 8 stale
  `staged_session` links accumulating (Phase 9 cleanup).

---

## 🟡 2026-06-18: Confirm-card fixes from live test (build-green, re-verify pending)
Phone lib now installed on Base44 (npm install done; resolves). Fixed from live test:
1. **Saved `3108904603` with no +1** — field was pre-filled a non-E.164 value → globe/no country →
   emitted raw digits. Fix: `StagedConfirmCard` only pre-fills phone if already E.164.
2. **Save allowed incomplete number** — now phone optional but validated via `isPossiblePhoneNumber`;
   "Save & continue" disabled until name set AND phone empty-or-valid. (Decision: optional, not
   hard-required — don't lock out a no-mobile band member.)
3. **No flag / cramped selector** — added `phoneNumberField.css` (wider boxed selector, bigger flag);
   `international` shows the dial code in the input.
4. **"Hey, jones_adamd"** → staged identity adopts confirmed first/last name; greeting now
   **"Hey {FirstName}"** (no leading comma).
- Serenova `0.6.0509`→**`0.6.0510`**. Build green.
- **Data:** cleared `jones_adamd` stale User.phone_mobile + membership flag/phones → clean re-test
  (one-off API PUT; guarded write script unchanged).
- **Re-verify:** card shows US flag by default, +1 prefixes the number, can't save a country-less
  number, MobileHub greets "Hey {FirstName}".

---

## 🟡 2026-06-18: Staged confirm-card phone → E.164 + country dropdown (build-green, verify pending)
B3 follow-up: owner entered `3108904603` (no +1); Twilio SMS (C2) needs E.164. Added
`react-phone-number-input` (`^3.4.17`) → new `src/components/ui/PhoneNumberField.jsx` (flag/country
dropdown, outputs E.164, **defaults to device locale region** = home country, not geolocation — a
touring artist abroad still has their home +1). Wired into `StagedConfirmCard`; `saveStagedProfile`
strips formatting as a backstop. Serenova `0.6.0508`→**`0.6.0509`**. Build green (lib + CSS bundled).
- **⚠️ `jones_adamd`'s saved `3108904603`** needs re-saving via the card/profile → `+13108904603`
  before C2 SMS testing (or normalize it).
- The other ~8 plain-text phone inputs in the app (MyInformationForm, EditUserDialog, team modals…)
  can adopt `PhoneNumberField` later for consistency — logged, not done.

---

## 🟡 2026-06-18: Mobile view-routing fix + Band/Crew login button (build-green, live verify pending)
Owner: "PWA on Home Screen always opens Desktop View on mobile." Two bugs fixed:
1. `preferDesktop` flag never expired (one "Switch to Web View" tap pinned the device to desktop
   forever); 2. the mobile→MobileHub redirect was only on Dashboard (mgmt users on ManagementDashboard
   never redirected).
- **New `src/lib/deviceView.js`** — `isMobileDevice`/`preferDesktopActive`/`setPreferDesktopView`
  (1-hour TTL via `preferDesktopUntil`)/`clearPreferDesktopView`. **Owner chose 1h.**
- **Redirect centralized in `Layout.jsx`** (covers all desktop pages incl. ManagementDashboard);
  removed the Dashboard-only copy. Deploying immediately un-sticks already-pinned devices.
- **`MobileHamburgerMenu`** Switch-to-Web-View → 1h preference; logout clears it.
- **`Login.jsx`** new "Band & Crew Sign In" button → `/StagedLogin` (closes discoverability gap).
- Serenova version bumped `0.6.0507`→**`0.6.0508`**. Build green.
- **⚠️ Verify:** mobile login lands on MobileHub; Switch to Web View holds desktop ~1h then reverts;
  Band/Crew button reaches StagedLogin; login screen shows v0.6.0508.

---

## 🟡 2026-06-18: Phase SA B3 BUILT (build-green, ⚠️ live verify pending)
First-login confirm card. Changed files (pushed): `src/components/mobile/StagedConfirmCard.jsx` (NEW),
`base44/functions/saveStagedProfile/entry.ts` (NEW), `base44/functions/getStagedMobileData/entry.ts`
(now returns `profile{first_name,last_name,phone_mobile,confirmed}`), `base44/entities/Membership.jsonc`
(+`phone_mobile`, +`staged_profile_confirmed_at`), `src/api/stagedSession.js` (+`saveStagedProfile`),
`src/pages/MobileHub.jsx` (captures `profile`, renders the card when `!confirmed`).
- **Flow:** first staged login → card (name + mobile #, pre-filled) → Save → `saveStagedProfile` writes
  to ALL the identity's memberships (`pending_profile_data` for display + top-level `phone_mobile` for
  C2's SMS lookup) + stamps `staged_profile_confirmed_at` → MobileHub. Re-shows until actually saved
  (persistent flag, not session count). Phone harvest = the C2 SMS prerequisite.
- **⚠️ Verify live:** deploy (Membership got new fields — additive, but confirm they provision); first
  login shows card, Save persists, second login skips it, phone lands on the membership row
  (`node scripts/base44.mjs Membership '{"user_email":"jones_adamd@me.com"}'` → check `phone_mobile` +
  `staged_profile_confirmed_at`). NOTE: `jones_adamd@me.com` already has a session → may read as
  already-confirmed=false (no flag yet) so the card SHOULD show on next login.
- **Next:** B4 (My Profile + 🔒 data-inventory gate), C2 (SMS — can now use harvested `phone_mobile`).

---

## ✅ 2026-06-18: Phase SA B2.1 VERIFIED LIVE
Tested as `jones_adamd@me.com` staged: contact **names resolve**, **Set List** + **Venue contacts**
visible, **no financials**, **no edit** anywhere. "Switch to Web View" correctly bounces back to
MobileHub (auth gate confines staged → `/MobileHub`; hard-cap holds at the routing layer too). B2.1
closed. **Logged-not-fixed (minor):** "Switch to Web View" is a no-op for staged (could hide it);
`MobileShareDialog` may still client-read when opened. **Next Phase SA:** B3 (first-login confirm card
→ capture mobile #), email deliverability (codes → iCloud spam — daytime, fresh non-iCloud test addr),
C2 (SMS channel), M (Twilio cost card).

## 🟢 2026-06-18 PM: Versioning system (per-hub) + deploy canary BUILT (build-green)
Owner wanted a real versioning system + every major hub versioned independently + all versioning
logged in a DB. Built (decisions log v2.75):
- **`src/version.js`** = source of truth (CODE constant — travels with the bundle, so a changed number
  on screen PROVES a deploy; bumping it = the small change that forces a Base44 rebuild). Registry:
  Serenova `0.6.0506` (main AMH) · System `1.0.0009` · Venue `0.5.0018` · Contract `0.4.0010` ·
  Financial `0.3.0007` · Storage `0.2.0003`. Format `v{major}.{minor}.{build4}`. **Bump the relevant
  hub here each deploy.**
- **`src/components/VersionStamp.jsx`** renders `v…` per hub. Shown: all password-auth screens (via
  `AuthLayout`), `StagedLogin`, **Layout bottom-left above username**, **MobileHub hamburger menu**
  (all = Serenova version); each hub page header shows its own (SystemHub, Contracts, Venues,
  FinancialHub, Storage).
- **DB audit log (best-effort, never the displayed value):** new `AppVersion` entity +
  `recordAppVersion` service-role fn (create-if-missing per hub+version), fire-and-forget from
  `App.jsx` on mount. ⚠️ **`AppVersion` entity must provision on Base44** (watch for the same
  non-provisioning issue `StagedSession` hit — display works regardless).
- **Use this as the deploy canary:** after a deploy, check the login screen shows the bumped number.
  Build green; all six version strings verified present in the built bundle.

---

## 🟢 LATEST — 2026-06-17 (night): Phase 5-M step 13 + Phase SA (Staged Passwordless Login) BUILT & VERIFIED

> Long session, all pushed to `main` (latest `6a774a6`), build green each time
> (`BASE44_LEGACY_SDK_IMPORTS=true npx vite build`). Repo docs current: build phases **v2.44**,
> decisions log **v2.72**, plus `docs/STAGED-ACCESS-WORKING-DOC.md`.

### ⚠️ DEPLOY GOTCHA got WORSE this session — read before testing anything
Base44 **"Synced" ≠ deployed**, and this session **reopen AND Publish both repeatedly failed to
pull pushed commits into Base44's working copy** (commit showed as Synced; served bundle hash never
changed; new functions/entities not live). **What reliably forced a deploy:** open any file in
**Base44's own code editor, add a space, Save** → it resyncs the working copy + rebuilds (expect a
"Loading your app" pause). Also: Base44's 2-way sync auto-commits back to GitHub, so `git pull
--rebase` before each push (hit one trivial conflict, resolved). Memory `base44-repo-sync` updated.

### Phase 5-M step 13 — DONE (commit `2c99608`)
Removed the two stray direct memberships in Lisa's artist acct `764957794471`
(`adam@originalartists.com` owner + `oa@originalartists.com` tour_manager) so mgmt access flows via
the **Original Artists LinkedAccount** (both resolve to `admin` via seat→level; verified). New
guarded write tool **`scripts/base44-write.mjs`** (DELETE-only, allowlisted, dry-run default).
**Team.jsx fix:** the Admin tab now **synthesizes link-only manager cards** (active LinkedAccount →
company seats → ManagerArtistAssignment when event-scoped) so the AMC group shows without a direct
membership. ⚠️ Live UI re-verify pending (Team Admin tab shows Original Artists group again).

### Phase SA — Staged Passwordless Login (NEW track) — A0 + B1 + B2 ✅ VERIFIED LIVE
**Goal:** band/crew reach **MobileHub** with no password — enter email → **6-digit code** (Twilio
Verify) → in-app → staged session keyed to their existing `staged`/`active` membership. **Decided:
codes not magic-links** (PWA-safe), **Twilio Verify for both email + SMS**, **email-first**.
- **Twilio is set up & PAID** (owner). Secrets in Base44: `TWILIO_API_KEY_SID`,
  `TWILIO_API_KEY_SECRET`, `TWILIO_VERIFY_SERVICE_SID` (Verify-restricted key: verification +
  verification-check Create only). Verify **Email** channel runs on **SendGrid** (domain
  `serenovahub.com` authenticated). **Twilio Verify is EXEMPT from A2P 10DLC** for OTP — SMS needs
  NO carrier registration, just the paid account. (Earlier "10DLC" worry was wrong.)
- **A0 — Admin Twilio test panel** (SystemHub → Cost & Usage → "Test function"): system-admin
  pop-up that fires a test email/SMS code (bypasses membership check). ✅ both channels send+verify.
  Backend `sendTwilioVerifyTest` + dialog in `CostUsageAdmin.jsx`.
- **B1 — login/session/auth gate** ✅: `src/api/stagedSession.js` (token in localStorage + seam),
  `src/pages/StagedLogin.jsx` (email→code screen), `App.jsx` gate confines staged → `/MobileHub`
  (additive; only runs for logged-out users → no lockout), MobileHub resolves staged identity,
  logout clears the token.
- **B2 — data path + hard-cap** ✅: service-role **`getStagedMobileData`** returns the staged user's
  events/travel/accom/tours/flights (mobile-safe; NO financials) — staged users can't do client
  `base44.entities` reads (no Base44 auth). **Hard-cap in `MobileHubView`:** staged = view-only,
  areas limited to events/travel/accom/setlist/repertoire, `canEdit=false`, REGARDLESS of role
  (tested: `jones_adamd@me.com` is an admin but the staged view must be view-only — ⚠️ eyeball not
  yet done). 
- **🔑 ROOT-CAUSE FIX:** the new `StagedSession` entity **wasn't persisting service-role writes**
  (0 rows despite ok → `invalid_session` 401 → "No account available"). Same-RLS
  `CompanyAccessCache` had rows → it was the freshly-added entity not provisioning during the rough
  deploys, NOT RLS. **Session token now lives on the proven `ShareableLink` entity**
  (`link_type:'staged_session'`, email in `entity_id`/`created_by_email`, `created_date`→30d TTL).
  `verifyStagedCode`/`validateStagedSession`/`getStagedMobileData` all use it. `StagedSession.jsonc`
  left unused (Phase 9 cleanup). Verified: a `staged_session` ShareableLink row persists; MobileHub
  loads events for the staged user.

### 🟡 2026-06-18: B2.1 BUILT (build-green, ⚠️ NOT yet live-verified)
Staged event-detail sub-data now flows through the service role. Changed files (not yet pushed/deployed):
- `base44/functions/getStagedMobileData/entry.ts` — also returns `setLists`, `venues`,
  `venueSnapshots`, and `memberProfiles` (`{email→{user,membership}}` roster name/contact map scoped
  to the account's roster emails; mobile-safe, still NO financials).
- `src/pages/MobileHub.jsx` — captures those into a `stagedExtras` state (initial load + artist
  switch) and threads `staged`+`stagedExtras` into `MobileEventDetail`.
- `src/components/mobile/MobileEventDetail.jsx` — in staged mode derives setlists/venue/snapshots/
  contacts from `stagedExtras` instead of client `base44.entities` reads. **Hard-cap leak fixed:** it
  called `usePermissionContext().canEdit` directly, so a staged admin would've gotten setlist Add/Edit
  + all booking refs — now `canEditSetlist`/`canSeeAllRefs` forced `false` when staged.
- **Out of scope, logged:** `MobileShareDialog` (event detail's Share button) may still client-read
  when opened by a staged user — evaluate whether Share should show for staged at all.

### 🔜 NEXT (Phase SA, in priority order)
1. **Verify B2.1 live** — deploy + sign in as `jones_adamd@me.com` staged; confirm in event detail:
   contact **names** resolve (no "Unknown User"), **Set List** shows, **Venue** info/contacts populate,
   and everything is **view-only** (no setlist Add/Edit buttons, no all-booking refs).
2. **Eyeball the broader hard-cap** as `jones_adamd@me.com` staged: confirm view-only / no edit / no financials.
3. **B3 — first-login confirm card** (required-once; capture **mobile #** for SMS login).
4. **Email deliverability (daytime task):** codes land in **iCloud/me.com SPAM** (new sending
   domain, no reputation, rapid repeat sends). Fix: confirm SendGrid domain auth fully Verified +
   add **DMARC** + reputation warmup. NOT a code bug — Twilio shows "Code sent via EMAIL".
   **Don't re-hammer `jones_adamd@me.com`** (rate-limited/burned); use a fresh non-iCloud test email.
5. **C2 — SMS channel:** enable `channel:'sms'` + add phone→membership lookup + phone capture. No 10DLC.
6. **M — SystemHub Twilio/SendGrid cost card:** LIVE actuals via **Twilio Usage Records API** (owner:
   "if it can pull, then it is true and live"); needs an added Twilio Usage-read credential.

### Tools / data notes
- `scripts/base44.mjs <Entity> '<queryJSON>'` = GET-only Base44 reads (api_key bypasses RLS).
- `scripts/base44-write.mjs` = guarded DELETE-only (allowlisted ids).
- Staged session check: `node scripts/base44.mjs ShareableLink '{"link_type":"staged_session"}'`.
- `jones_adamd@me.com` = active **admin** membership in Lisa's acct `764957794471` (the staged test
  identity — admin on purpose, to test the hard-cap).

---

## 🟢 2026-06-16/17: Event wizard, Team/Settings rebuild, role model + Phase 5-M

> Long session. All pushed to `main`, build green each time (`BASE44_LEGACY_SDK_IMPORTS=true npx vite build`).
> Latest commit `7e4e948`. Full detail in `docs/SERENOVA-DECISIONS-LOG.md` (v2.67) +
> `docs/SERENOVA-BUILD-PHASES.md` (v2.39) + `docs/SERENOVA-ARTIST-MGMT-ACCOUNT-USER-PERMISSIONS.md`.

**⚠️ CRITICAL GOTCHA (cost real time):** Base44 **"Synced" in the feed ≠ deployed.** A push
only syncs git; the served bundle rebuilds **only when the owner fully reopens the Base44
project**. Symptoms of stale build: incognito shows no change, inspected DOM matches old code.
After any push, tell the owner to **reopen the project**, then hard-reload. (Memory `base44-repo-sync` updated.)

### Events — Create/Import "wizard everywhere" (Phase A + B)
- **EditEvent was a gutted stub** (silently reduced 427→237 lines on 2026-06-05 under a
  permission commit; the 7 tab components survived orphaned). **Restored** the tabbed editor
  (`EditEvent.jsx`): Event/Financial/Personnel/Schedule/Guest List/Advance/Action Items.
- **`src/components/events/EventWizard.jsx`** (NEW) — 4-step **Deal Info → Performances →
  Provisions → Contacts** for both manual create + PDF import. `CreateEvent.jsx` hosts it;
  `ImportEvent.jsx` passes the uploaded contract. Billing-derived event name (strips share %),
  VenueHub venue picker, **Management & Booking** group (Artist Mgmt + Tour Manager + Agency +
  typeable Agent + Band/Lineup picker → roster), **Buyouts** (category/amount/note),
  **Exclusivity** (before/after radius, mi/km toggle), Money & Terms, contract saved to
  `contract_files` + status logic (attach → confirmed; one party → Executed, both → Complete).
  Performances: Date / Set Time / **Duration (min)** / **Doors (optional, blank)**.
- **`extractEventFromPDF`** schema expanded (agency/agent/is_agent_deal, billing, multi-perf +
  duration, provisions, event_contacts, deal_notes, exclusivity). Extraction quality is
  AI-dependent per contract.
- **Event.jsonc additive fields:** `booking_agent_name`, `is_agent_deal`, `exclusivity{}`,
  `day_of_show_contact_ids[]`, `roster_members[].is_lead`, `performances[].doors_time`.

### Venue contacts unified + bugs fixed
- **Single source of truth = `Venue.contacts[]`** (each now has a stable **`id`**). Events
  **link by id** (`Event.day_of_show_contact_ids`, **live reference**). `DayOfContactsCard`
  rewritten to link model; `VenueDashboard` reads embedded contacts; wizard contacts step
  links/creates venue contacts. New `src/components/venues/venueContactUtils.js`.
- **"137 contacts (auto-synced from AMC)" bug fixed** — `auto_populated` wasn't a persisted
  field, so the AMC team got re-appended every load; now dedupe by email + field added.

### Team + Settings rebuilt; EditUserDialog fixed
- **Team.jsx restored + v2:** tabs **Admin / Band / Crew / Groups**; click a member →
  `EditUserDialog` (staged=full edit; verified=account-scoped only, email always locked;
  `User.update` gated to self → fixed the "not authorized" save). Band-group **integrity
  flags** (red ⚠ for non-member / non-canonical instrument). Canonical instruments =
  `src/constants/rolesAndInstruments.js`. **Admin tab grouped by linked company** (Owner →
  per-AMC groups → Account Team). Personnel: names from first/last, instrument shows, lead→band.
- **Settings consolidation:** `src/pages/Settings.jsx` merges Account Settings + Account Admin
  (General/Print/Representation/Permissions/Billing); old pages route to it; nav = Profile ·
  Settings · Team. Fixed "No account selected" (used `Account.filter`, not `.get` by record id)
  + added `Account.print_settings`.
- **Bottom-left user area = clickable menu** (My Profile / Account Settings / All Accounts /
  System Hub / Log Out).

### Role / ownership model — FINALIZED (spec: `SERENOVA-ARTIST-MGMT-ACCOUNT-USER-PERMISSIONS.md`)
- **Artist = Owner always** (`Account.owner_user_email`/`artist_user_email`), even if a mgmt
  company set up the account. **Management company = full Admin via the `LinkedAccount`** (Admin/
  Owner seat → `artist_admin`), NOT a member "Owner" role. **Display/functional role ≠ access**:
  a mgmt person shows by their assigned functional role (e.g. Tour Manager) but accesses per seat.
- **Account claiming:** Artist always (+ BMC fallback); **Artist precedence**.
- **BMC** = financial write (deposit/balance/monies, invoices/payments, settlement notes, post-
  closeout with logging) + view (band/crew, contracts, riders, calendar, financials, receipts);
  no control. **Booking Agency** = its own **Booking Agency Hub** (no artist-account access;
  own events only; **Routing View** like the Kurland sheet). **One owner per account; bands have
  multiple admins.** `act_type` (solo/duo/trio/band) is the sub-type (fixed a mislabel).

### Phase 5-M progress (steps in build phases)
- ✅ 13 (authoritative ownership labels), 14 (functional-role display), 15 (Representation lists
  mgmt users — via auto-sync), 18 (context-aware switcher: artist=own accounts; mgmt=Return to
  Management + assigned artists), **10 (company-seat→artist access resolver — ADDITIVE)**,
  **14b (seat-role escalation in the resolver)**.
- **Resolver (`usePermissions.js`):** in the "no direct membership → zero access" branch, for an
  artist account, grants access via an active `LinkedAccount` to a mgmt company the user is
  seated in (scoped by `ManagerArtistAssignment` if `is_event_scoped`); seat role → artist
  role_level (`SEAT_TO_ARTIST_LEVEL`). STRICTLY ADDITIVE (direct-membership users + owners
  resolve earlier; can only grant) + fail-safe. Uses **Base44 entities** (Supabase memberships
  empty); `role_templates` maps level (artist levels: super_admin/admin/finance/tour_manager/
  band_member/crew/viewer).

### Routing / caching fixes
- **User-switch cache:** Layout sessionStorage cache was user-agnostic → "act as" kept the prior
  user; now verifies `auth.me().email` vs cached and clears+reloads on mismatch. (Reverted an
  over-correction that forced `/`→Entry and bounced to Onboarding/AccountSelection mid-switch.)
- **Management user → ManagementDashboard** guard in `Dashboard.jsx`; blue company box links home;
  fixed the "Select an account" **splash flash** (gate loader on `contextLoading`).

### Link previews (OG)
- `index.html` had NO Open Graph tags → iMessage showed the big square icon. Added OG/Twitter +
  wide `lockup-on-dark.png` banner; `src/utils/pageMeta.js` sets per-page title (SharedEventView →
  event name). Crawler dynamic-title is best-effort (Base44 prerender); server share endpoint = follow-up.

### 🔜 NEXT / OPEN
1. **Phase 5-M step 13 cleanup (the main open item):** remove the **stray direct `owner`
   memberships** for `adam@originalartists.com` + `oa@originalartists.com` in Lisa's account
   (`764957794471`) so their access flows via the link (now safe: step 10+14b give admin via the
   Original Artists link). **Blocked:** `scripts/base44.mjs` is GET-only → do via the Team UI
   (remove member) or a write path. **Then VERIFY** the link-only path (no link-only user exists
   yet — both mgmt users have direct memberships, so step 10/14b is shipped but unverified live).
2. New phases logged, not started: **Phase FC (Event Closeout & expanded Expenses)**,
   **Booking Agency Hub + Routing View**, **BMC Hub**, **Account Administration & Billing Core**,
   default-account ("star") routing. Plus prior tracks (EPK, Staged Access, Repertoire, day-sheet).
3. Watch items: ~25 Base44 entities still use broken `user_condition.role` RLS (Venue +
   AccountBilling fixed) — audit if "saves but doesn't stick." Extraction prompt tuning per contract.

---

## 🟢 2026-06-12 PM: MobileHub features + brand + auth/perm fixes

**Two live auth/perm fixes (decisions log 2026-06-12T11:30):** (1) **auth gate** (`3603e20`, `App.jsx`) — logged-out users now `<Navigate to="/login">` (was missing → protected routes/AccountSelection hung); logout handlers → `/login`. (2) **account-owner full access** (`3698c0e`, `usePermissions.js`) — artist owner resolved to ALL-none (role `artist` unmapped); added an **owner bypass** (email === account `owner_user_email`/`artist_user_email` → full) + `artist`→`super_admin`. Caveat: if an owner still gets none, the account record's owner/artist email isn't set (Base44 data fix).

**MobileHub feature pass (all pushed):**
- **Add to Home Screen / PWA** (`c569ac5`): `public/manifest.json` + brand icons, `useInstallPrompt` hook, menu item (Android one-tap / iOS instructions / hidden once installed). **LIVE-ONLY** on `app.serenovahub.com` (not the Base44 iframe) — confirmed working by owner.
- **Hotel popup inside event detail** (`c569ac5`); **mobile setlist editor** (`5fe83cf`, `MobileSetListEditor` — inline, line-type icons, ↑/↓ reorder, repertoire picker; replaces web SetListDialog on mobile).
- **Fixes** (`72315f2`): stray "0" in hotel rooming was `{room.rate && …}` rendering falsy 0 → guarded `>0`; flight passenger confirmation moved to a right-aligned badge.
- **Day-sheet home** (`f1264bd`): "Tonight's hotel" card at the bottom (under event+schedule, per owner's hotel-placement note). **Day-sheet redesign is the NEXT thing to continue** (owner paused 2026-06-12 PM). Future: "Mine vs All", Now/Next hero, pull-to-refresh.

**Brand (`59ec504` icons / `e60a28e` re-skin):** real Serenova icon set app-wide (favicons, PWA, apple-touch, Layout header mark). MobileHub "accent + chrome" re-skin: header→**teal `#284854`**, bg→**cream `#faf8f4`**, borders→**sage `#b8b8aa`**, avatar→**tan**, nav-active→teal; functional colors kept. Palette + tan-on-light rule in memory `serenova-brand-palette`. **⚠️ OPEN:** owner unsure on the teal header — offered **charcoal `#232c27`** (darker, on-brand) as a one-line swap to try next. Body text still neutral slate (could → charcoal). Palette is **MobileHub-only** (not the web app yet).

---

## 🟢 2026-06-12 AM: MobileHub polish + two live auth/perm fixes

**MobileHub travel-UX + visuals polish** (commits `fb0c7d0`/`6610273`/`b41594f`): bolder card borders + darker page bg; event-view Flights/Ground cards now slim + clickable → `MobileTravelDetailDrawer` (with **Recommended Airport Arrival** 2h/3h); home schedule + cards show **traveller** (First L) + **booking ref**, scoped to travel editors (`canEdit('travel')`) + the individual traveller (own booking). See `docs/MOBILEHUB-WORKING-DOC.md` "Polish pass".

**Two live auth/permission fixes (Lisa Fischer debugging) — decisions log 2026-06-12T11:30:**
- **Auth gate** (`3603e20`, `src/App.jsx` + logout handlers): logged-out users now redirect to `/login`. Was missing — `AuthenticatedApp` only redirected on an explicit `authError`; a no-token session (`isAuthenticated=false`, no error) rendered protected routes / hung on AccountSelection "loading", and logout didn't reach login. Logout handlers → `/login`.
- **Account-owner full access** (`3698c0e`, `usePermissions.js`): artist owner resolved to ALL-none because her membership role `artist` wasn't in `BASE44_ROLE_TO_LEVEL` (→ no template → none). Added an **account-owner bypass** (email === account `owner_user_email`, or `artist_user_email` for artist accounts → full, even with no membership) + mapped role **`artist`→`super_admin`**. **Caveat:** if an owner still gets none, the account's `owner_user_email`/`artist_user_email` isn't their email AND role≠`artist` (Base44 data fix).
- Google-vs-email provider is **undeterminable from our side** (Base44-internal; `User` entity has no provider field; Supabase auth empty) → Base44 admin Users panel.

---

## 🟢 Feature-build session (2026-06-11 PM): MobileHub + guests + EPK/Staged scoping

Owner direction locked (memory `serenova-build-strategy`): **keep building on Base44, clean up as we go, keep the permission bridge; defer Phase 0/F.** New rule: **every new module = a new canonical permission area** (a gate on a non-existent area = deny-all, the cause of the morning lockout). Base44↔repo: pull before editing, push = deploy ("open Base44 to deploy"), never edit the same file in the builder + locally at once.

**Shipped this session (all pushed to `main`, build green each time):**
- **MobileHub permission gating** (`5503cdf`) — was routed OUTSIDE Layout with NO gating. Split into outer (data + a local `AppContext` + `PermissionProvider` keyed to the active account) + inner view; tabs/home-widgets hide by area (events/travel/accommodations). Mgmt-user-on-linked-artist (no direct membership) → all tabs (no regression).
- **MobileHub read enrichment** (`2bbff37`, `4e34b9b`) — travel folded into Today's Schedule; flight booking ref on cards; hotel check-in/out **times** added; venue **parking** + **Venue Contacts** moved to Venue tab, **Contacts = band/crew only**. *(Discovery: the flight/hotel drawers were already very rich.)*
- **Setlist editing on mobile** (`f95cce4`) — reuses the real `SetListDialog`, gated `setlist` edit.
- **NEW canonical area `guest_list` (20th)** (`9b50f81`) — migration `role_templates_add_perm_guest_list` (owner/admin=full, manager/tour_manager=edit, **band_member=`own`**, else none). CLAUDE.md 19→20 areas.
- **Guest-list bug + model fix** (`5dd5573`) — root cause: guest lists keyed by `performance_date` ALONE → same-day shows shared one list. Now keyed by **date+time** (legacy date-only matched + upgraded) + comp-limit model (over `total_comps` → auto **"Requested"** needing admin/artist confirm; `own` = members request only). **Wired** the (previously orphaned) `GuestListTab` into **EditEvent's guest_list tab** (`7750078`) with a guest-scoped save. ⚠️ **Discovery: `EditEvent.jsx` is a migration STUB** (name/venue/promoter only); rich event editing may warrant its own rebuild track. Owner wants guest editing later as an **EventDetails button+popup** (deferred).
- **Mobile Band & Crew contacts fixed** (`70a9487`, `5da613d`) — names showed as email-prefix + no phone because `roster_members` store only emails. Now **enriches from User + Membership** (via `getUsersByEmailsForAccount` + `Membership.filter`, same join as PersonnelTab) → real name via `getDisplayName` (KnownAs > First Last) + **phone + email**.

**Scoped (docs only — not built):**
- **EPK** (`docs/EPK-WORKING-DOC.md`, `773adec`) — public shareable webpage; tokenized sharing EXTENDS existing band-view `ShareableLink` tokens (owner correction — not new auth); artist-set access modes (open/one-time/email-gated 7-day); scoped/curated sends; multiple/grouped EPKs; quotes (groups + 3 defaults); open/geo/email tracking.
- **Staged Access** (`docs/STAGED-ACCESS-WORKING-DOC.md`, `e217a6b`) — passwordless **magic-link** → staged session → **MobileHub-only**, view/own only, never financials/admin. Email=identity (swaps MobileHub's identity source). **Crew=staged-only** (same-email port later); **band members=staged + free-account upsell**. **Unblocks the deferred band-member mobile writes** (guest/expense requests). Shares an **email-token primitive with EPK**. Decoupled from Phase 0/F. Phases A–E in the doc.

**Queued next (in `docs/MOBILEHUB-WORKING-DOC.md` + the scoping docs):** mobile **band-member guest-request flow** + **expense submission** (Group 2c/2d — depend on Staged Access for non-account identity); guest editing → **EventDetails popup**; view-only/token-page contact scoping (name-only); **Staged Access Phase A** (email-token primitive); EPK build. Also still open: FinancialHub **closeout** (status-field unify + wire the Lock button + seed `settlement`) — audited, ~80% built.

**Working docs are the source of truth:** `MOBILEHUB-WORKING-DOC.md`, `EPK-WORKING-DOC.md`, `STAGED-ACCESS-WORKING-DOC.md`, `FINANCIALHUB-WORKING-DOC.md`. Canonical areas now **20**.

---

## 🔴 Permission lockout hotfix + canonical-area cleanup (2026-06-11T17:05)

Artist **owner** `ellephish@gmail.com` (Lisa Fischer account) was **locked out** (only Dashboard/Mobile Hub; `400 (memberships)` → `[usePermissions] resolution error`). Root-caused live + fixed. Commits `b5cdd52` (Tier 0), `3ec21c0` (Tier 1), `a3ef365` (Tier 2), `4960420` (docs). Build green; **no schema/data change**.

- **Tier 0 — the unlock (`src/hooks/usePermissions.js`):** Supabase `memberships.user_id`/`account_id` are **`uuid`** but auth is Base44 (non-uuid ids) → query `400`ed and `throw memErr` aborted resolution to the catch **without setting `permissions`** → `can()`=`none` everywhere; Base44 fallback skipped. Fix: `isUuid()` guard skips the Supabase memberships + entity-grant queries for Base44 ids; **any** query error now falls through to the Base44 fallback (never throws); `role_templates` error degrades to no-access. **Owner bypass broadened** from `account_type==='system'` to **any `role_level==='super_admin'`** → account owners get `full` regardless of template (the live `(artist,super_admin)` template had repertoire/setlist/members=`none`).
- **Tier 1 — dead gates:** `admin`/`schedule` aren't canonical areas (+ `manage` isn't a level) → deny-all. AccountSettings/ManagementAdmin→`settings`, AccountAdmin→`members`, DatabaseBackup→`settings`/`full`, Schedule→`events`; `RoleTemplate.js` import → `@/api/supabaseClient`. No `role_templates` migration.
- **Tier 2 — nav keys:** canonicalized nav `permissionKey`s (`press_kit`→`press`, `venues`→`events`, `reports`→`financials`, Storage→`storage`, `account_admin`→`members`, `account_settings`→`settings`, Mgmt Admin→`settings`, Artist Roster→`team`). Same nav groups are ALSO filtered by `buildArtistCan` (legacy Base44 template keys) in the mgmt-viewing-artist context → added **`NAV_AREA_ALIASES`** so it resolves both legacy + canonical (no regression). `Dashboard.jsx` → canonical `usePermissionContext`/`PermissionGate`.
- **Deferred (logged):** `companyAccess.jsx` `getDefaultPermissionsByRole` keeps legacy `financial_hub`/`press_kit` keys — it **mirrors the Base44 template `.permissions` shape** consumed as `[entity][action]`; canonicalize with the Phase 9 template removal. **Artist Roster→`team`** is a best-fit guess (owner to confirm). Artist **non-owner** template tuning (e.g. artist `admin` repertoire/setlist=`none`) deferred. Real fix for the uuid mismatch = **Phase 0/F**.
- **⚠️ Verify on next Base44 open:** re-test as `ellephish` — full left-nav + pages render, no `400 (memberships)`; and as a band_member — repertoire/setlist editable, financials hidden (bypass didn't over-grant).

---

## The Repo

**Code:** [jonesadamd/serenovahub_b44](https://github.com/jonesadamd/serenovahub_b44) — `main` branch
**Docs:** `/docs` folder in `serenovahub_b44` is the single source of truth for all decisions, architecture, and build status. Artifact files shared in chat threads are considered stale.

---

## What Serenova Hub Is

A SaaS platform for independent touring artists and their management teams. Multi-account, multi-role. Core modules: Events/Tours, Travel, Accommodations, Financials, Contracts, Riders, Press Kit, Setlists, Archive, Storage.

Tech stack: React + Vite, Base44 (BaaS/hosting), Supabase (DB + auth), Tailwind, Lucide icons.

---

## Environment Variables — Supabase API Keys

**All Supabase credentials live in `.env.local` (never committed).**

| Variable | Format | Status |
|---|---|---|
| `VITE_SUPABASE_URL` | `https://<project>.supabase.co` | ✅ **Required** |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | `sb_publishable_...` | ✅ **Required — use this** |
| `VITE_SUPABASE_ANON_KEY` | `eyJ...` (JWT) | ❌ **Legacy — do not use** |

### ⚠️ Critical — Do NOT use `ANON_KEY`

This project uses `VITE_SUPABASE_PUBLISHABLE_KEY` (`sb_publishable_...` format).
Supabase deprecated the `anon`/`public` JWT key in favour of publishable keys.
`VITE_SUPABASE_ANON_KEY` is **not used anywhere in this codebase**.

- Canonical Supabase client: **`src/api/supabaseClient.js`** — import from here everywhere
- If you see `ANON_KEY` anywhere in the codebase, replace it with `PUBLISHABLE_KEY`
- Reference: https://supabase.com/docs/guides/auth/publishable-keys

**`.env.local` template:**
```
VITE_SUPABASE_URL=https://<your-project-ref>.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=sb_publishable_<your-key>
```

---

## Current Phase

**Phase 1 — Permission System & Auth Foundation — ✅ COMPLETE**
**Phase 2 — Events & Tours — ✅ COMPLETE**
**Phase 3 — Travel & Accommodations — ✅ EFFECTIVELY CLOSED OUT (2026-06-11T09:18 EDT, commit `9511e38`): Steps 1–3 ✅, Step 2 follow-up ✅ (gating + `require` standardized), entity-grant query safe-fixed. Only the grant-based event-narrowing remains deferred.**
**Pre-Phase-4 broken imports — ✅ FIXED (2026-06-11T09:31 EDT, commit `4fc2f05`; correction #3 below).**
**Phase 4 (Contract Management) — 🔄 IN PROGRESS (2026-06-11T11:07 EDT, commits `62c88a8`→`5aedb1a`).** Step 1 audit found `CreateContract`/`EditContract` were **phantom** (never built) + a 3-schema divergence + that the **Supabase data layer isn't writable** (no Supabase auth session; `auth.users`/`public.users` empty; `contracts`/`events`/`tours`/`entity_access_grants` RLS-on/zero-policies = deny-all; `usePermissions.js` imports `useCurrentUser` from a non-existent `@/api/entities`). **Decision:** build on **Base44 behind a seam** (`src/api/contracts.js`), lean core + sync scaffolding. **Built:** extended `Contract.jsonc`; seam; `CreateContract`/`EditContract`/`ImportContract` (shell) pages; fixed `Contracts.jsx` list; registered routes. **Step 7 ✅ DONE (commit `04d6383`)** — Base44 contract↔tour linking (Linked Event auto-derives tour from `events.tour_id`; Linked Tour picker; list shows tour label). Supabase `event_tour_links` → Phase 9. **Phase 4 core (Steps 1–7) COMPLETE; ContractHub/AgencyHub depth deferred.**
**Phase 5 (Financial Hub) — ✅ COMPLETE 2026-06-11 (Steps 1–3, permission gating):** audit (all Base44; gating via `canEdit` prop) → Step 2 gated 3 ungated-write Settings components (`CommissionSettings`/`TeamRateSettings`/`FinancialDefaults`, commit `db12474`) → Step 3 settlement-area gating in `EventFinancialDetails` (`canSettle = canEdit('settlement')`; `deal_memo` has no UI surface, commit `8c3590e`). **`docs/FINANCIALHUB-WORKING-DOC.md` = living source of truth** for the FinancialHub **v1 product build** (settlement engine, versioned rate library, invoicing, document attachments, status workflow, **per-artist/deal commission model**: gross/net/net-less-authorized + role comp + rate structures) — a separate large track coupled to Phase 0/F + Phase 9.
**Phase 6 (Press/Riders/Setlists) — ✅ GATING COMPLETE 2026-06-11 (Steps 1–2, commit `22c7493`):** all Base44; `Riders` already gated; fixed `press_kit`→`press` bug (PressKit/PressKits); gated `SetListDialog` setlist writes; `SetLists` orphan wrapped on `setlist`. **NEW canonical area `repertoire`** (18th) added — `PERM_AREAS` + `role_templates.perm_repertoire` migration (seeded from `perm_setlist`); `Repertoire.jsx` (song library, already used `area="repertoire"`) now resolves. **Repertoire feature track** (owner vision): unify Songs + setlists + **usage tracking over time** — Supabase-native build, Phase 0/F-coupled. SystemHub Base44 template editor still needs a `repertoire` module (Phase 5-M step 12).
**Phase 7 (BandGroups & Team) — ✅ GATING COMPLETE 2026-06-11 (Steps 1–2, commit `cb4b925`):** **BandGroup is live on Base44** (not stubbed — CLAUDE.md corrected). Fixed `Team.jsx` `require="manage"` no-op bug → `edit`; **surfaced `BandGroupsManager` on the Team page** for artist accounts (artist/admin/management create their own groups), writes gated `canEdit('team')`; `InviteUserForm` gated `canEdit('members')`. **Deferred:** Supabase `band_groups` table + BandGroup migration → Phase 0/F + 9; `team_assignments` → Phase 5-M. NOTE non-canonical levels (`manage` fixed, `members`'s `invite`) to reconcile later.
**Phase 8 (Archive & Storage) — ✅ GATING COMPLETE 2026-06-11 (Steps 1–2, commit `42a4e4a`):** all Base44 (`Event`/`StorageFile`); `Archive` gated correctly (`archive`). Fixed non-canonical `area="storage"` bug by adding a **dedicated `storage` area** (19th — owner-chosen; `PERM_AREAS` + `role_templates.perm_storage` migration seeded from `perm_archive`); `Storage`/`StorageHub` already used `area="storage"` → now resolve; `StorageFileTable` delete gated. **Storage/Files feature track** (owner vision): expand StorageHub (uploads/search/**file↔event linkability incl. old events**/recording sessions); **event+version-scoped band-member file access** (setlists/scores for events they were on, version they were privy to) — ties to entity-grant/event-narrowing (Phase 3 Step 3 + 0/F) + Repertoire. `StorageQuotaEditor` → system-gate refinement.

> **GATING PHASES DONE:** Phases 2–8 permission gating all complete. Canonical areas now **19** (added `repertoire`, `storage`). What remains is the big foundation + feature builds: **Phase 0/F** (Supabase auth+RLS — unblocks everything), **Phase 9** (Supabase migration), **Phase 5-M** (Mgmt templates), **Phase S** (security), and the product tracks (ContractHub, FinancialHub v1, Repertoire, Storage/Files).

---

## ⚡ 2026-06-11 PM SESSION — major events after 11:07 (read this — lots changed)

> Full detail in `docs/SERENOVA-DECISIONS-LOG.md` (v2.28) and `docs/SERENOVA-BUILD-PHASES.md` (v2.17). Repo HEAD `638ec0c`.

1. **App was crash-looping → now renders.** Fixed a chain of import/runtime crashes (commits `00649c4`, `f1da600`, `5bc5720`): `PrintEventItinerary`→`base44.entities`; `Membership.js` shadow → `membershipConstants.js` + a `Membership.js` re-export; `usePermissions`/`usePermissionContext` **compat shims** for 13 legacy files; **`useCurrentUser`** local hook (was importing a phantom `@/api/entities` → ProxyObject crash); membership-query **PGRST201** ambiguous-embed fix.
2. **CI gate LIVE + green** (`.github/workflows/ci.yml`, commit `49e7e63`): `npm ci` + `BASE44_LEGACY_SDK_IMPORTS=true npx vite build` on push/PR. The legacy flag is required because ~60 files still use `@/entities/*` (Phase 9 migration). Standalone `vite build` now passes.
3. **Permission system — Base44-sourced fallback BRIDGE** (commit `f731926`): Supabase `memberships`/`users`/`accounts` are **EMPTY** (only `role_templates`=28). So `usePermissions` now, when no Supabase membership, derives `(account_type, role_level, custom_permissions)` from the **Base44** membership (AppContext) + a `BASE44_ROLE_TO_LEVEL` map, and resolves via Supabase `role_templates`. **Migration applied:** `role_templates_public_read` RLS policy (was deny-all; reference data, read-only). Tour manager etc. now get real per-role access. BRIDGE — remove at Phase 0/F.
4. **`.env.local` security** — owner added `VITE_SUPABASE_*` to Base44 platform; rotation + untracking **deferred to Phase S (Security Hardening)** (secrets newly created). Not untracked (would break the build).
5. **Artist-Management permissions spec adopted** — owner doc renamed `docs/SERENOVA-ARTIST-MGMT-ACCOUNT-USER-PERMISSIONS.md` = **source of truth** (7 templates, 9 roles, company modules + artist-access elements). Gap analysis → **Phase 5-M steps 3, 8–12** (template content, company-seat→artist resolver, event-scoping by assignment, trailing read-only bug, two-template-system reconciliation, SystemHub admin access). NOT built — scheduled.
6. **Shim & Bridge Cleanup Register** (build phases) tracks all 10 temporary bridges from today with removal conditions (Phase 0/F / Phase 9).
7. **NEW phases in roadmap:** **Phase 0/F** (Supabase Auth session + RLS foundation — pre-Phase-9 prerequisite) and **Phase S** (Security Hardening).

> **Permission reality (important):** the app runs on Base44 for auth + data; Supabase has only `role_templates` populated + RLS policies mostly deny-all. The canonical Supabase permission system works ONLY via the Base44 bridge until Phase 0/F migrates users/accounts/memberships into Supabase.
> ⚠️ **NEW pre-Phase-9 prerequisite — "Supabase Auth session + RLS foundation"** (build phases "Phase 0/F"): bridge Base44 auth → a Supabase session (`auth.uid()` resolves), author per-table RLS policies, fix the `@/api/entities` import, and migrate the ~11 users (all in Base44; Supabase users tables empty → export, create `auth.users`+`public.users`, link `auth_user_id`, password-reset/Google on first login). Phase 9 silently assumed this. Contract pages all go through `src/api/contracts.js` — the one file that swaps to Supabase when this lands.

> ⚠️ **READ FIRST — corrections from the 2026-06-11 full-project verification:**
> 1. **`CreateTravel.jsx` does not exist** — it was a phantom in prior audits. The real wizard entry page is **`Travel.jsx`**. All references below to `CreateTravel` should be read as `Travel.jsx`.
> 2. **`PagePermissionGuard.jsx` canonical-import fix applied** (commit `3da5b1c`) — it previously imported the legacy `utils/PermissionContext`, whose provider is never mounted; all 26 guard-wrapped pages depended on a non-existent context.
> 3. **Broken imports — ✅ FIXED 2026-06-11T09:31 EDT (commit `4fc2f05`).** `Contracts.jsx`, `FinancialHub.jsx`, `EditEvent.jsx`, `EventFinancialDetails.jsx` now use canonical `usePermissionContext` + default `PagePermissionGuard`/`PermissionGate`; `hasPermission(area,level)` mapped to `canView`/`canEdit`; non-canonical `financial_hub`/`finance` → `financials`. `Contracts.jsx` loads cleanly → Phase 4 clear to start. **Remaining (Phase 9):** `EditEvent.jsx` `guest_list` tab (no canonical area / no panel) left gated-closed; `EditEvent` still heavily Base44; legacy `utils/permissionChecks.jsx` still broken. See decisions log 2026-06-11T09:31 EDT.
> 4. **SECURITY:** `.env.local` is COMMITTED to the repo with `SUPABASE_SECRET_KEY`, Resend, and R2 secrets. Rotation + untracking pending owner action — see decisions log. Do not print its contents.
> 5. **`CLAUDE.md` now exists at repo root** — Claude Code sessions auto-load it; keep it in sync with this handoff.
> 6. A vanilla `vite build` fails on pre-existing `PrintEventItinerary.jsx` absolute import (`/src/entities/Event`) — Base44-plugin-dependent; fix required for off-Base44 migration.
> 7. **Entity-grant system — query SAFE-FIXED 2026-06-11T09:18 EDT (commit `9511e38`), still dormant.** `usePermissions.js` now queries the real columns (`access_level, entity_type`, account-scoped) and resolves grants via `ENTITY_TYPE_TO_AREA` (event→events, travel→travel, accommodation→accommodations, contract→contracts), upgrade-only. The `entity_access_grants` table is `user_id, account_id, entity_type, entity_id, access_level, granted_by, expires_at` — one access_level per entity, **no per-area column**. Table is still EMPTY with no population UI, so the path is dormant; grant-based event-narrowing in `Travel.jsx` stays deferred (`accessibleEventIds = null`). The legacy `event_access_grants` table / `memberships.is_event_scoped` column do not exist live. **STILL BROKEN (untouched, Phase 9 sweep):** legacy `utils/permissionChecks.jsx` `resolveUserLevel()` reads non-existent `entity_access_grants.area/level`, `memberships.is_owner/user_email/permission_template_id`, and a `role_template_areas` table. See decisions log 2026-06-11T09:18 EDT.

### Phase 1 — All Steps Complete

| Step | Description | Status |
|---|---|---|
| 1 | Define `PERM_LEVELS`, `PERM_AREAS`, `canAtLeast()` | ✅ Done |
| 2 | Build `usePermissions` hook — full 6-level resolution | ✅ Done |
| 3 | Create `Membership.js` and `RoleTemplate.js` entity files | ✅ Done |
| 4 | Fix `PermissionContext.jsx` (BUG-01) | ✅ Done |
| 5 | Fix `PermissionGate.jsx` (WARN-01) — Rules of Hooks fix | ✅ Done |
| 6 | Extract `LEVELS` + `meetsLevel()` to `permissionUtils.js` | ✅ Done |
| 7 | Wire permission gating into Layout nav | ✅ Done |
| 8 | Gate `Events`, `FinancialHub`, `Contracts` pages | ✅ Done |
| 9 | Create `src/api/supabaseClient.js` — missed Phase 1 prerequisite; fixed bad import in `usePermissions.js` | ✅ Done |

### Phase 2 — Events & Tours — All Steps Complete

| Step | Description | Status |
|---|---|---|
| 1 | Confirm `Events.jsx` reads from `events` table — audit complete; DDL applied (`event_type`, `promoter`) | ✅ Done |
| 2 | `Events.jsx` search aligned to canonical `title` field — dual `title`/`name` check with Phase 9 inline comment | ✅ Done |
| 3 pre-work | Audit `CreateEvent.jsx` + `EditEvent.jsx` — Base44 only confirmed, no Supabase client, full entity inventory logged | ✅ Done |
| 3 | Gate `CreateEvent` / `EditEvent` behind `canEdit('events')` — `CreateEvent` already correct; `PagePermissionGuard` outer wrapper added to `EditEvent` | ✅ Done |
| 4 | Gate `EventFinancialDetails` behind `canView('financials')` — `withPermission` HOC applied at default export; inline `useEffect` guard retained. Commit: `0d1cc54` | ✅ Done |
| 5 | `event_tour_links` DDL — table confirmed in Supabase with full spec. `id`, `event_id` (FK→events CASCADE), `tour_id` (FK→tours CASCADE), `account_id` (FK→accounts CASCADE), `created_at`. UNIQUE `(event_id, tour_id)`. RLS + `account_isolation` policy. Indexes on `event_id`, `tour_id`, `account_id`. | ✅ Done |
| 6 | Tour status flow: 7-value dropdown (`draft/planning/confirmed/in_progress/in_closeout/completed/cancelled`) in `CreateTour.jsx` and `TourOverviewTab.jsx` STATUS_COLORS. `in_closeout` badge = purple. Commits: migration `tours_add_status_column`; code commit `49b9952`. | ✅ Done |

### Phase 3 — Travel & Accommodations

| Step | Description | Status |
|---|---|---|
| **1** | **Audit flight booking wizard (`CreateTravel → AddTravel → EditFlightBooking`)** | **✅ Complete** |
| **2** | **Gate travel writes — DONE 2026-06-11T08:11 EDT (commits `3da5b1c`, `78b49f1`, `1d529b6`): guard canonical-import fix; `Travel.jsx` broken-import fix (no page guard — preserves own-itinerary fallback); `AddTravel` + `EditFlightBooking` wrapped `area="travel" require="view"` with `canEdit('travel')` gates on all write handlers** | **✅ Complete** |
| 2 follow-up | `ImportTravel.jsx` + `EditTravel.jsx` ungated (scope decision pending); standardize `require="view"` vs `"edit"` for Add/Edit pages at Phase 3 close-out | ⬜ |
| **3** | **Co-travel visibility — DONE 2026-06-11T08:54 EDT (commit `e095f33`): own-itinerary filter now applied to FLIGHTS in `Travel.jsx` (parallel to the pre-existing segment filter — flights previously had none, so band/crew saw every account flight); co-traveler names visible; confirmation numbers shown for the user's OWN booking only (flights + both segment legs), via a single `sanitizeConfirmation` helper** | **✅ Complete** |
| 2 follow-up | **DONE 2026-06-11T09:18 EDT (commit `9511e38`):** `ImportTravel`/`EditTravel` gated; `require` standardized **Direction A** — read pages `view`, write pages `edit`. `ImportTravel`/`EditTravel`/`AddTravel`/`EditFlightBooking` all `area="travel" require="edit"` (AddTravel + EditFlightBooking moved `view`→`edit`). Accommodation Add/Edit already `edit`; lists stay `view`. | ✅ Complete |
| Infra fix | **DONE 2026-06-11T09:18 EDT:** `usePermissions.js` entity-grant query safe-fixed (schema-correct, dormant — correction #7). | ✅ Complete |
| 3 follow-up | Entity-scoped event **narrowing** still deferred — `accessibleEventIds` stays `null`. Now blocked only on an undesigned grant model + empty table / no population UI (the column-name bug is fixed). Own sub-step once grants can actually be written. | ⬜ |

#### Phase 3 Step 2 — Progress Detail

| Sub-task | Status |
|---|---|
| Tour status filter fix — `AddTravel.jsx` `loadTours()` | ✅ Done (2026-06-10T19:22 EDT) |
| Tour status filter — `EditFlightBooking.jsx` | ✅ Already correct — no change needed |
| `PagePermissionGuard.jsx` canonical-import fix (prerequisite for all gating) | ✅ Done (2026-06-11T08:11 EDT) |
| `Travel.jsx` — broken legacy import fixed; `canView('travel')`; no page guard (own-itinerary fallback preserved for Step 3) | ✅ Done (2026-06-11T08:11 EDT) |
| `PagePermissionGuard area="travel" require="view"` on `AddTravel`, `EditFlightBooking` | ✅ Done (2026-06-11T08:11 EDT) |
| `canEdit('travel')` gates — `handleSaveFlight` (AddTravel); `handleSave`/`handleDelete`/`saveTravelerEdits`/`removeTravelerFromGroup`/`separateTraveler` (EditFlightBooking) | ✅ Done (2026-06-11T08:11 EDT) |

#### Phase 3 Step 3 — Progress Detail (commit `e095f33`, 2026-06-11T08:54 EDT)

**Scope: `Travel.jsx` only.** Owner-confirmed decisions: (1) co-traveler names shown; (2) confirmation numbers shown for own booking only; (3) entity-grant event narrowing deferred.

| Change | Status |
|---|---|
| `canViewAllTravel` (`isSystemUser \|\| canView('travel')`) hoisted above flight processing; `sanitizeConfirmation(email, value)` helper added | ✅ Done |
| Flights: own-itinerary `.filter()` on flight groups for non-view users (mirrors the existing segment filter; flights previously had NO narrowing) | ✅ Done |
| Confirmation numbers routed through `sanitizeConfirmation` — flights + both segment legs (own shown, co-travelers nulled) | ✅ Done |
| `accessibleEventIds` kept `null`; comment updated to record the deferral | ✅ Done |

**Deferred (Step 3 follow-up):** entity-scoped event narrowing — blocked on the `usePermissions.js` `entity_access_grants` column-name bug (correction #7) and an empty table with no population UI. **Known limitation:** per-flight `item.bookings` still holds co-travelers' raw `booking_reference` in client memory (not rendered; delete/unlink use `booking.id` and are perm-gated). True data-layer enforcement is a Phase 9 Supabase-RLS concern.

---

## Phase 3 Step 1 — Audit Findings (2026-06-09)

### Files Read
- `src/pages/CreateTravel.jsx`
- `src/pages/AddTravel.jsx` (54KB — largest travel file)
- `src/pages/EditFlightBooking.jsx`

### What the Wizard Does
The flight booking wizard is a 3-page flow:
1. **`CreateTravel`** → entry point; routes to `AddTravel` or accommodation create
2. **`AddTravel`** → full flight booking creation: flight search, PNR entry, traveler selection, cost, event/tour linking, expense allocation
3. **`EditFlightBooking`** → edits a single `EventFlightBooking` record + its `FlightBookingGroup`; supports PNR separation, per-traveler seat/class edits, segment management via `PnrSegmentManager`

### Data Sources — All Still Base44
All three files use **only `base44.entities.*`** — no Supabase reads or writes anywhere in the wizard.

Key entities used:
- `base44.entities.EventFlightBooking` — per-traveler booking record
- `base44.entities.FlightBookingGroup` — PNR-level group (cost, allocation, links)
- `base44.entities.Flight` — individual flight segments
- `base44.entities.Event` — linked events (for context/expense)
- `base44.entities.Tour` — linked tours (filter fixed in Step 2 — see below)
- `base44.entities.Membership` — team member list for traveler selector
- `base44.functions.invoke('getUsersByEmailsForAccount')` — enriches membership with display names
- `base44.entities.FlightSearchResult` — used in AddTravel for flight lookup

### Permission Issues
- `EditFlightBooking` imports `useAppContext` correctly ✅
- No permission gating at all on any travel page — **Step 2 remaining work**
- No wrong-file permission imports (unlike `TourDetails.jsx` / `CreateTour.jsx`)

### Tour Status Filter — RESOLVED ✅
`AddTravel.jsx` `loadTours()` had old filter `['planning', 'confirmed', 'in_progress']` — **fixed 2026-06-10T19:22 EDT** to `['draft', 'planning', 'confirmed', 'in_progress', 'in_closeout']`.
`EditFlightBooking.jsx` already had the correct 5-value filter — no change needed.

### AddTravel — Additional Flags
- Uses `base44.entities.FlightSearchResult` — unknown if this has a Supabase equivalent; flag for Phase 9
- `expense_allocation_type` options: `event`, `tour`, `split_events`, `paid_by_venue` — well-structured, no changes needed

### EditFlightBooking — Booking Status Values
`EventFlightBooking.booking_status` dropdown: `planned / booked / checked_in / completed / cancelled` — these are **flight booking statuses** (not tour statuses) and are correct as-is.

### What Step 2 Still Needs (remaining)
1. Add `PagePermissionGuard area="travel" require="view"` wrapper to `CreateTravel`, `AddTravel`, `EditFlightBooking`
2. Add `canEdit('travel')` check before any create/update/delete call in all three files

---

## Phase 3 Step 2 Pre-Work Audit — TourDetails.jsx + CreateTour.jsx (2026-06-10)

### Files Read
- `src/pages/TourDetails.jsx`
- `src/pages/CreateTour.jsx`

### TourDetails.jsx — Findings
- **Data source:** Fully Base44-native — `base44.entities.Tour.get(id)`, events via `base44.entities.Event.filter({ id: { $in: tourData.event_ids } })` (denormalized `event_ids` array on Tour entity, not `event_tour_links`)
- **Permission hook import:** Uses `usePermissions` from `utils/PermissionContext` (wrong file) — Phase 9 scope, defer
- **Permission area name:** `hasPermission('financial_hub', 'view')` — wrong area name; canonical is `financials` — Phase 9 scope, defer
- **`canEdit` check:** `hasPermission('events', 'edit')` — correct area, wrong hook import

### CreateTour.jsx — Findings
- **Data source:** Fully Base44-native — `base44.entities.Tour.create(tourData)`, writes `event_ids: []` directly onto Tour record
- **Event linking:** After create, writes `base44.entities.Event.update(eventId, { tour_id: newTour.id })` back to each event; available events filtered by `e => !e.tour_id` (reads `events.tour_id` Base44 field)
- **Status dropdown:** Confirmed 7 values (`draft/planning/confirmed/in_progress/in_closeout/completed/cancelled`) with default `draft` — ✅ already correct from Phase 2 Step 6
- **Permission hook import:** Same wrong hook import (`utils/PermissionContext`) — Phase 9 scope, defer

### Step 2 Scope — What This Changes
Neither `TourDetails.jsx` nor `CreateTour.jsx` are in scope for Step 2. Their permission hook import issues and `event_tour_links` migration are Phase 9 work. No changes to these files in Step 2.

### Deferred to Phase 9 (from this audit)
| Issue | File | Decision |
|---|---|---|
| Wrong permission hook import (`utils/PermissionContext`) | `TourDetails.jsx`, `CreateTour.jsx` | Phase 9 rewrite |
| `hasPermission('financial_hub', ...)` wrong area name | `TourDetails.jsx` | Phase 9 rewrite |
| Event linking via `tour.event_ids` array (not `event_tour_links`) | Both files | Phase 9 rewrite |
| `events.tour_id` filter in `loadAvailableEvents` | `CreateTour.jsx` | Phase 9 rewrite |

---

## Booking Agency Architecture — Decision (2026-06-10)

**Option C selected:** Booking Agency is a **separate product (Serenova Agency)** connecting to Serenova Hub via a public API. It is NOT built inside Serenova Hub.

**What's already in Supabase (prerequisite DDL — applied 2026-06-10):**
- `booking_agency` added to `account_type` enum ✅
- `artist_assignments` table created ✅

**`artist_assignments` schema:**
```sql
CREATE TABLE artist_assignments (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  artist_account_id   uuid NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  member_user_id      uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  company_account_id  uuid NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  assignment_type     text NOT NULL CHECK (assignment_type IN (
                        'artist_management',
                        'business_management',
                        'booking_agency'
                      )),
  assigned_at         timestamptz DEFAULT now(),
  assigned_by         uuid REFERENCES users(id),
  UNIQUE (artist_account_id, member_user_id, assignment_type)
);
```

**`artist_assignments` vs Phase 9 `companies`/`company_staff`/`company_artist_assignments` — NOT in conflict:**
- `artist_assignments` → links **Serenova auth users** from a company account to a specific artist account (drives dropdowns + API auth)
- `companies` / `company_staff` / `company_artist_assignments` → external company contact directory (Phase 9, replaces Base44 `ManagementCompany` / `BookingAgency` entities)

**What is deferred to Phase 10 / Serenova Agency product:**
- Full booking agency dashboard, contracting flow, offer pipeline
- Promoter CRM, routing / availability views
- Public API design, OAuth/token authorization scaffold
- Serenova Agency codebase (separate repo)

**`artist_admin_company` = Business Management Team account type** — no new account type needed. UI label is "Business Management"; `account_type` value stays `artist_admin_company`.

---

## Phase 5-M — Management & Multi-Account (Key Steps)

> Prerequisite DDL complete: `artist_assignments` table + `booking_agency` account_type applied 2026-06-10.

| Step | Description | Status |
|---|---|---|
| Pre-DDL | `artist_assignments` table + `booking_agency` enum value — migration applied ✅ | ✅ Done |
| 1 | Management Dashboard roster view | ⬜ |
| 2 | Company seat invite + termination flow | ⬜ |
| 3 | Trailing access enforcement on `access_scoped_until` | ⬜ |
| 4 | `ManagementTeam.jsx` — member list with assigned artists display per member | ⬜ |
| 5 | `ManagementAdmin.jsx` — artist roster × team assignment matrix; assign/unassign UI writing to `artist_assignments` | ⬜ |
| 6 | Dropdown filtering — wire `artist_assignments` query into company-type selectors; confirm general pickers exclude management/agency staff | ⬜ |
| 7 | Management company pricing model (see `SERENOVA-PRICING-MODEL.md`) | ⬜ |

---

## Permission System Architecture (Current — Supabase-native)

```
Layout.jsx
  └─ <PermissionProvider accountId={currentAccount?.account_id}>
       └─ calls usePermissions({ accountId })  ← one Supabase query
            └─ broadcasts via PermissionContext
                 └─ consumers:
                      usePermissionContext()       ← zero extra queries
                      <PermissionGate area="..." require="...">
                      useCanAccess('area', 'level')
                      withPermission(Component, 'area')
```

**6 permission levels (ordered):** `none → view → own → export → edit → full`

**17 permission areas:** `financials, contracts, archive, events, travel, team, press, riders, billing, settings, tech_pack, stage_plot, settlement, deal_memo, setlist, accommodations, members`

**Resolution order (highest wins):**
1. system `super_admin` → full bypass
2. `access_status` guard → inactive/suspended = zero access
3. role template (`role_templates` table)
4. custom overrides (`memberships.custom_permissions` JSONB)
5. entity-scoped grants (`entity_access_grants` table, upgrade-only)
6. temporal expiry check

**Page-level gating pattern (all confirmed pages following this):**
```jsx
export default function PageName() {
  return (
    <PagePermissionGuard area="<area>" require="view">
      <PageNameContent />
    </PagePermissionGuard>
  );
}
```

**withPermission HOC (component-level gating — used for EventFinancialDetails):**
```jsx
// Defined at bottom of src/components/PermissionContext.jsx
export function withPermission(Component, area, requiredLevel = 'view', fallback = null)

// Usage (as applied to EventFinancialDetails):
export default withPermission(EventFinancialDetails, 'financials', 'view');
```

**Key files:**
- `src/hooks/usePermissions.js` — core hook
- `src/components/PermissionContext.jsx` — provider + `usePermissionContext()` + `withPermission()` HOC
- `src/components/PermissionGate.jsx` — `<PermissionGate>` + `useCanAccess()`
- `src/components/permissions/PagePermissionGuard.jsx` — page-level guard
- `src/utils/permissionUtils.js` — `LEVELS`, `meetsLevel()`
- `src/Layout.jsx` — single file (Base44 constraint), contains all nav logic
- `src/api/supabaseClient.js` — **canonical Supabase client** — import from here everywhere

**DO NOT revert to:**
- `base44.entities.PermissionRoleTemplate` (legacy)
- `base44.entities.UserPermissionOverride` (legacy)
- old dot-notation `perms['area.action'] === true`
- `src/components/utils/PermissionContext.jsx` (wrong file, different API)
- `VITE_SUPABASE_ANON_KEY` (legacy — use `VITE_SUPABASE_PUBLISHABLE_KEY`)

---

## Supabase Project

**Project name:** `serenova-hub-production`
**Project ID:** `jpoharsecyebkhsnbnfe`
**Region:** `us-east-1`
**Status:** ACTIVE_HEALTHY

### Tables — Confirmed Present in `public` schema (all RLS enabled)

| Table | Status |
|---|---|
| `events` | ✅ Full schema confirmed live (23 columns, direct query 2026-06-10T19:22 EDT) — see schema below |
| `memberships` | ✅ Present |
| `tours` | ✅ Present (`status` column added — 7 values: `draft/planning/confirmed/in_progress/in_closeout/completed/cancelled`, default `draft`) |
| `event_tour_links` | ✅ Schema-ready — FK cascade on `event_id`, `tour_id`, `account_id`; UNIQUE `(event_id, tour_id)`; RLS `account_isolation` policy |
| `event_venue_details` | ✅ Present |
| `team_assignments` | ✅ Present |
| `contracts` | ✅ Present |
| `performance_ticket_links` | ✅ Present |
| `artist_assignments` | ✅ Present — links Serenova auth users from company accounts to artist accounts; migration `artist_assignments_and_booking_agency_type` applied 2026-06-10 |
| `band_groups` | ❌ **Not yet created — deferred to Phase 7** |

### `events` Table — Full Confirmed Schema (23 columns, queried 2026-06-10T19:22 EDT)

| # | Column | Data Type | Nullable | Default |
|---|---|---|---|---|
| 1 | `id` | `uuid` | NO | `gen_random_uuid()` |
| 2 | `account_id` | `uuid` | NO | — |
| 3 | `tour_id` | `uuid` | YES | — |
| 4 | `contract_id` | `uuid` | YES | — |
| 5 | `title` | `text` | NO | — |
| 6 | `status` | `event_status` enum | YES | `'inquiry'` |
| 7 | `event_date` | `date` | YES | — |
| 8 | `event_time` | `time` | YES | — |
| 9 | `end_date` | `date` | YES | — |
| 10 | `is_multi_day` | `boolean` | YES | `false` |
| 11 | `door_time` | `time` | YES | — |
| 12 | `set_length_minutes` | `integer` | YES | — |
| 13 | `num_shows` | `integer` | YES | `1` |
| 14 | `venue_name` | `text` | YES | — |
| 15 | `venue_city` | `text` | YES | — |
| 16 | `venue_country` | `text` | YES | `'US'` |
| 17 | `notes` | `text` | YES | — |
| 18 | `metadata` | `jsonb` | YES | `'{}'` |
| 19 | `created_by` | `uuid` | YES | — |
| 20 | `created_at` | `timestamptz` | YES | `now()` |
| 21 | `updated_at` | `timestamptz` | YES | `now()` |
| 22 | `event_type` | `text` | YES | — |
| 23 | `promoter` | `text` | YES | — |

**Key schema notes:**
- `events.tour_id` IS a real Supabase column (nullable uuid) — not Base44-only
- `events.timezone` does NOT exist — add via migration at Phase 9 (or earlier if Phase 3 requires it)
- `events.name` does NOT exist — canonical display field is `title`
- Travel wizard pickers need `id`, `title`, `event_date`, `venue_city` — all present ✅

### Known Schema Gaps (Phase 9 work — no DDL until then)
- `events.status` enum missing `Prospect/Tentative` — Base44 uses this; Supabase has `inquiry` as closest match. Decision deferred to Phase 9.
- `events` canonical field is `title`; Base44 entity uses `name`. Phase 9 task: rename `form.name` → `form.title` in CreateEvent.
- `EditEvent.jsx` uses `localStorage.getItem('currentAccountId')` in two places — replace with `AppContext.currentAccount.account_id` at Phase 9 rewrite.
- `base44.functions.invoke('autoLinkCompaniesToEvent')` in EditEvent — needs Supabase-native equivalent at Phase 9.

### Tables still needing DDL (before Phase 9)
- `event_tasks`
- `event_user_assignments`
- `event_venue_snapshots`
- `booking_agencies`
- `companies`, `company_staff`, `company_artist_assignments` (Phase 9 — external company contact directory)

---

## Key Architecture Rules

- `Layout.jsx` stays as a **single file** — Base44 platform constraint, accepted tech debt
- No `localStorage` / `sessionStorage` for permissions — sandboxed iframe constraint
- All routing changes must update `pages.config.js`
- `memberships` table replaces old `account_members` table
- Entity grants are **upgrade-only** — never downgrade below role template floor
- Financial Hub has 3 render tiers: `full` (edit+), `limited` (view), `none` (redirect)
- **BandGroup** — `base44.entities.BandGroup` calls in `Events.jsx` are stubbed as `[]` until Phase 7 when `band_groups` table is created. Do not create the table early.
- **`artist_assignments`** — assignment is a UX/display layer only. It does not grant or restrict permissions. Permissions remain governed by `memberships` + `role_templates` + `entity_access_grants`.

---

## Base44 Session Rules (Strictly Enforced)

1. Read affected files before modifying — no assumptions
2. Confirm exact files and change list before executing any push
3. No "while I was there" changes — scope only
4. All decisions logged in `docs/SERENOVA-DECISIONS-LOG.md` with `YYYY-MM-DDTHH:MM EDT`
5. All phase completions updated in `docs/SERENOVA-BUILD-PHASES.md`
6. All doc updates must include `Last Updated: YYYY-MM-DDTHH:MM EDT` timestamp
7. Boring is correct — no cleverness in infrastructure

---

## Docs Index (serenovahub_b44/docs/)

| File | Purpose |
|---|---|
| `README.md` | Quick-start index + open issues table + **env var reference** |
| `SERENOVA-PERMISSION-SYSTEM.md` | Permission deep-dive, resolution logic, Financial Hub tiers |
| `SERENOVA-BUILD-PHASES.md` | Phase-by-phase checklist with status |
| `SERENOVA-DECISIONS-LOG.md` | All architectural + product decisions with timestamps |
| `SERENOVA-TECHNICAL-DOCS.md` | CSS tokens, routing, DB config, Base44 rules, changelog |
| `SERENOVA-HUB-DOCS-2.md` | Master reference — account model, schema, pages |
| `SERENOVA-ACCESS-RIGHTS-MIGRATION-PLAN.md` | Access rights matrix, termination rules |
| `SERENOVA-PRICING-MODEL.md` | Subscription tiers, seat model, Stripe plan |
| `SERENOVA-BASE44-ENTITY-REFERENCE.md` | All 56 Base44 entities mapped to phases 2–9 |

---

## Space Instructions (for Perplexity Space config)

```
Repo: jonesadamd/serenovahub_b44 (main branch)
Docs: /docs folder in repo is single source of truth — always read before making changes.
Session memory: jonesadamd/perplexity_project_memory — SERENOVA-SESSION-HANDOFF.md

Rules:
1. Read affected files before modifying
2. Confirm exact changes before any push
3. No scope creep — changes only what was agreed
4. Log all decisions in docs/SERENOVA-DECISIONS-LOG.md (YYYY-MM-DDTHH:MM EDT)
5. Update docs/SERENOVA-BUILD-PHASES.md on step completion
6. Layout.jsx stays single file (Base44 constraint)
7. Permission system is Supabase-native — never revert to Base44 entities
8. Supabase client key = VITE_SUPABASE_PUBLISHABLE_KEY — ANON_KEY is legacy, do not use
```
