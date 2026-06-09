# Serenova Hub — Session Handoff
> Quick-load context for any new AI session working on this project.
> **Always read this first. Then read the repo docs before touching any code.**
> Last Updated: 2026-06-09T12:56 EDT

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
**Phase 3 — Travel & Accommodations — 🔄 IN PROGRESS (Step 1 audit complete; Step 2 is next)**

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
| **2** | **Gate travel/accommodation writes; fix old tour status filter in AddTravel + EditFlightBooking** | **⬅ NEXT** |
| 3 | Co-travel visibility for band/crew (own itinerary only, no confirmation numbers) | ⬜ |

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
- `base44.entities.Tour` — linked tours (filter: `status: { $in: ['planning', 'confirmed', 'in_progress'] }` ← **uses old status values**)
- `base44.entities.Membership` — team member list for traveler selector
- `base44.functions.invoke('getUsersByEmailsForAccount')` — enriches membership with display names
- `base44.entities.FlightSearchResult` — used in AddTravel for flight lookup

### Permission Issues (Phase 9 scope — defer)
- `EditFlightBooking` imports `useAppContext` correctly ✅
- No permission gating at all on any travel page (Step 2 will fix this)
- No wrong-file permission imports (unlike `TourDetails.jsx` / `CreateTour.jsx`)

### Status Values Bug — CONFIRMED
`EditFlightBooking.jsx` and `AddTravel.jsx` both load tours with:
```js
base44.entities.Tour.filter({ status: { $in: ['planning', 'confirmed', 'in_progress'] } })
```
These are the **old pre-Phase-2-Step-6 status values**. The canonical values are now `draft/planning/confirmed/in_progress/in_closeout/completed/cancelled`. This filter will return **zero active-lifecycle tours** for any tour in `draft` or `in_closeout` status. **Must fix in Step 2:** Replace with `['draft', 'planning', 'confirmed', 'in_progress', 'in_closeout']`.

### AddTravel — Additional Flags
- Uses `base44.entities.FlightSearchResult` — unknown if this has a Supabase equivalent; flag for Phase 9
- `expense_allocation_type` options: `event`, `tour`, `split_events`, `paid_by_venue` — well-structured, no changes needed

### EditFlightBooking — Booking Status Values
`EventFlightBooking.booking_status` dropdown: `planned / booked / checked_in / completed / cancelled` — these are **flight booking statuses** (not tour statuses) and are correct as-is.

### What Step 2 Must Change
1. Add `PagePermissionGuard area="travel" require="view"` wrapper to `CreateTravel`, `AddTravel`, `EditFlightBooking`
2. Add `canEdit('travel')` check before any create/update/delete call in all three files
3. Fix Tour filter in `EditFlightBooking` and `AddTravel`: `['planning', 'confirmed', 'in_progress']` → `['draft', 'planning', 'confirmed', 'in_progress', 'in_closeout']`

---

## Phase 3 Step 2 Pre-Work Audit — TourDetails.jsx + CreateTour.jsx (2026-06-09)

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
| `events` | ✅ Schema-ready (`event_type`, `promoter`, `title` confirmed) |
| `memberships` | ✅ Present |
| `tours` | ✅ Present (`status` column added — 7 values: `draft/planning/confirmed/in_progress/in_closeout/completed/cancelled`, default `draft`) |
| `event_tour_links` | ✅ Schema-ready — FK cascade on `event_id`, `tour_id`, `account_id`; UNIQUE `(event_id, tour_id)`; RLS `account_isolation` policy |
| `event_venue_details` | ✅ Present |
| `team_assignments` | ✅ Present |
| `contracts` | ✅ Present |
| `performance_ticket_links` | ✅ Present |
| `band_groups` | ❌ **Not yet created — deferred to Phase 7** |

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

---

## Key Architecture Rules

- `Layout.jsx` stays as a **single file** — Base44 platform constraint, accepted tech debt
- No `localStorage` / `sessionStorage` for permissions — sandboxed iframe constraint
- All routing changes must update `pages.config.js`
- `memberships` table replaces old `account_members` table
- Entity grants are **upgrade-only** — never downgrade below role template floor
- Financial Hub has 3 render tiers: `full` (edit+), `limited` (view), `none` (redirect)
- **BandGroup** — `base44.entities.BandGroup` calls in `Events.jsx` are stubbed as `[]` until Phase 7 when `band_groups` table is created. Do not create the table early.

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
