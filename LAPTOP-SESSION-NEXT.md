# Laptop Session — Next Up (owner priorities, 2026-06-22 night → 2026-06-23)

> For the **laptop** machine. Read `SERENOVA-SESSION-HANDOFF.md` + `SERENOVA-MACHINE-SYNC.md` first,
> then this. Two items the owner wants to look at tonight/tomorrow. Claim each in the machine-sync
> Active-work table before building (desktop is idle; both are unclaimed).
> Created by the desktop session at `0.6.0575` (code HEAD `d537c5a`).

## Standing rules (don't skip)
- `git pull --rebase` BOTH repos before starting AND before each push.
- Build: `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build`. Bump `src/version.js` each deploy.
- Commit author **Adam Jones <aj@adamdcjones.com>** ONLY (NO Claude co-author trailer — owner owns authorship). Confirm the file list before any push.
- Base44 **"Synced" ≠ deployed** → force a rebuild via the editor add-space-Save; new functions/entity
  fields must **provision**.
- **PWA install / standalone / push only work LIVE on `app.serenovahub.com`** — NOT in the Base44 preview iframe.

---

## ITEM 1 — MobileHub home nudge: "Add to Home Screen" + "Enable notifications"

**Goal (owner):** a dismissible banner on the **MobileHub Home tab** that nudges the user:
- **(a) Not installed as a PWA** → prompt to **Add to Home Screen**.
- **(b) Installed (standalone) but push notifications NOT granted** → prompt to **Enable notifications**.
- Both **dismissible**, and **reappear every ~3 days** (owner: "maybe cancelled and reappear every 3 days or something").

**Reuse (already built — don't reinvent):**
- `src/hooks/useInstallPrompt.js` → `{ isIOS, isStandalone, canPromptInstall, promptInstall }`.
- `src/lib/push.js` → `isPushSupported()`, `pushPermission()` ('granted'|'denied'|'default'), `enablePush()`.
- `src/components/mobile/MobileHamburgerMenu.jsx` **already wires both actions** (`handleInstall`,
  `handlePush`, the iOS "Add to Home Screen" instruction sheet) — mirror that logic.

**Suggested approach:**
- New `src/components/mobile/MobileHomeNudge.jsx`, rendered at the **top of the Home tab** in
  `src/pages/MobileHub.jsx` (the `activeTab === "home"` block — above the guest-notice banners / week strip).
- Decision logic (show at most ONE):
  - `!isStandalone` → **install nudge**. Android/Chrome: `canPromptInstall` → `promptInstall()`. iOS:
    reuse the menu's Add-to-Home-Screen instruction sheet (no programmatic install on iOS Safari).
  - else if `isStandalone && isPushSupported() && pushPermission() !== 'granted'` → **enable-notifications
    nudge** → `enablePush()`.
  - else → render nothing.
- **Dismiss + 3-day reappear:** a localStorage key per nudge type (e.g. `nudge_install_until`,
  `nudge_push_until`) = `Date.now() + 3*86400e3`; suppress while `Date.now() < stored`. (Keep them
  separate so dismissing "install" doesn't hide "enable notifications" after they install.)
- **Gotchas:** live-only (see standing rules); iOS install = instructions only; push needs the deployed
  service worker `public/sw.js` (exists from Push Group 1). Don't nag if `pushPermission()==='denied'`
  (they actively blocked — a nudge can't re-prompt; maybe show a one-time "enable in settings" hint then stop).

---

## ITEM 2 — Live flight updates in MobileHub (Lisa flies 2026-06-23; she has the PWA ✅)

**Goal (owner):** surface **live flight status — delay / gate change / status** on the mobile flight
detail so Lisa sees it in action tomorrow. She's the **artist owner = logged-in** (not staged), so the
existing per-user live path works for her.

**Reuse (infra is ~all there):**
- **Web reference component:** `src/components/dashboard/LiveFlightTracker.jsx` — already consumes live
  flight data on the web dashboard. Mirror its fetch + display on mobile.
- **Backend fns:** `getUserUpcomingFlightsWithLiveData` (per-user; flights + cached live data),
  `fetchLiveFlightData` (fetch/refresh ONE flight via FlightAware AeroAPI — key `AEROAPI_KEY_LIVE`,
  cost-gated, only runs ~24h-before → 6h-after departure), `refreshLiveDataForUpcomingFlights` (batch,
  **MANUAL — no cron yet**), `acknowledgeFlightAlert`.
- **`LiveFlightData` entity:** `status` (enum incl. `delayed`), `departure_delay_minutes`,
  `arrival_delay_minutes`, `departure_gate`/terminal, estimated times, `alerts[]` (delay/gate_change/
  terminal_change/cancellation; severity; acknowledged).
- **Mobile surface:** `src/components/mobile/MobileTravelDetailDrawer.jsx` (the flight detail drawer —
  already shows times + "Recommended Airport Arrival"). Add a **live-status block** there.

**Suggested approach (smallest path to "see it tomorrow"):**
- When the mobile flight drawer opens for a flight **within the live window**, fetch live data
  **on-demand**: `fetchLiveFlightData({ flightId })` (or read via `getUserUpcomingFlightsWithLiveData`)
  → render a status chip: **On time / Delayed N min / Gate {x}**, the (updated) gate/terminal, and the
  estimated departure if delayed. Color the chip (green on-time / amber delayed / red cancelled).
- Lisa = logged-in → `base44.auth`-gated fns work. **Staged band/crew live flights = a follow-up**
  (route through a service fn like `getStagedMobileData`, since staged has no Base44 auth).
- **Verify before relying on it:** `AEROAPI_KEY_LIVE` is set + cost-gate allows it; Lisa's flight has a
  `Flight` record + `EventFlightBooking` so it shows up; her flight is inside the day-of window
  tomorrow (fetch returns live data only then).
- **No cron** means data is only as fresh as the last fetch — for tomorrow, on-demand-on-open is enough
  to demo it. The automated version (periodic refresh + **push** on delay/gate) = **Push Group 3**
  (separate task in the board; needs `NOTIFY_INTERNAL_SECRET` + a GH-Actions cron + `dispatchNotification`).

**This item is the natural precursor to Push Group 3** — build the on-demand live readout first (visible
win for Lisa), then layer the cron + push alerts.

---

## After either item
Bump `src/version.js`, build green, push (confirm file list w/ owner), update the machine-sync pointers +
mark the task done, add a decisions-log entry, and refresh the handoff. Deploy = editor add-space-Save
(+ provision any new function/entity). For Item 2, watch the AeroAPI cost (live key).
