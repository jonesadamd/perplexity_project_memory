# Serenova Hub — Session Handoff
> Quick-load context for any new AI session working on this project.
> **Always read this first. Then read the repo docs before touching any code.**
> Last Updated: 2026-06-18T19:00 EDT

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
