# Serenova Hub — Session Handoff
> Quick-load context for any new AI session working on this project.
> **Always read this first. Then read the repo docs before touching any code.**
> Last Updated: 2026-06-11T17:05 EDT

---

## 🔴 LATEST — Permission lockout hotfix + canonical-area cleanup (2026-06-11T17:05)

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
