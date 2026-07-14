# Serenova Hub — Machine Sync & Coordination

> **Read this right after `SERENOVA-SESSION-HANDOFF.md`.** Two machines (desktop + laptop) each
> run an AI coding session against the same code repo (`jonesadamd/serenovahub_b44`) and share
> this memory repo (`jonesadamd/perplexity_project_memory`) via git. This doc is the lightweight
> protocol that keeps the two sessions from clobbering each other or duplicating work.
> Last Updated: 2026-07-14T19:45 EDT
> **🖥️ DESKTOP wrapped 2026-07-14 evening. Repo `main` @ `18ffc17` = LIVE `0.7.0629`** (deployed +
> verified live). Full session: EventFinancialDetails test-and-fix pass + Guest List popup redesign +
> "Assigned to" picker fixes, three deploys (`0.7.0627`→`28`→`29`) — see **🔝 TOP BRIEF** in
> `SERENOVA-SESSION-HANDOFF.md` for the complete detail + the full unfinished-items list (Export/PDF/
> shareable-link, SystemHub Staged Users UI). **Owner moving to LAPTOP — opening in ~30 min.**
> **On the laptop FIRST: `git pull --rebase` BOTH repos**, then read the TOP BRIEF's "UNFINISHED"
> section — that's the priority-ordered pickup list, in the owner's own words.
> **AUTHORSHIP:** commits author **Adam Jones only — NO Claude trailer**. `LICENSE` + `SIGNATURE.md` (overt
> RUSHMERE/KABINGA markers, display-only, NEVER in security/randomness code) exist; no markers in code yet.

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
7. **Commits:** author **Adam Jones <aj@adamdcjones.com>** ONLY — **no Claude co-author trailer** (owner owns authorship; 2026-06-23).
   Confirm the file list before any push.

## ⚙️ Build / deploy — VIA THE BASE44 CLI (the editor "Publish" is BROKEN as of 2026-06-23)
> The editor's git auto-sync wedged + "Publish to live" ships a frozen build. **Deploy with the
> `base44` CLI.** Full detail: decisions log v2.149/v2.152 + the `serenova-deploy-test-flow` auto-memory.
- **Build:** `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite build` (plain `npx vite build` may
  grab the wrong vite). **`.env.production` (committed) MUST be present** — it bakes in `VITE_BASE44_APP_ID`
  / `APP_BASE_URL` / `BACKEND_URL`; without it the build 404s every `/api` call for fresh users. Verify:
  `grep -o 68c22e8ff3726c063c4a53e2 dist/assets/index-*.js` must match.
- **Deploy (owner runs — classifier blocks Claude from prod deploys):** `mv base44/entities ../_entities_HOLD`
  (the CLI's entity validator rejects the repo's `user_condition` RLS format) → build →
  `npx -y base44@0.0.56 --app-id 68c22e8ff3726c063c4a53e2 site deploy -y` → `mv ../_entities_HOLD base44/entities`.
  Functions: `… functions deploy <names…>` (NEVER `--force`).
- **Verify:** curl `https://app.serenovahub.com/?cb=$RANDOM` → check the `index-*.js` hash + grep the version.
- Bump the affected hub in `src/version.js` each deploy; the on-screen canary = source of truth for "deployed."

## 🔍 LOCAL PREVIEW — test changes BEFORE live (works on any machine)
> Established 2026-06-24 (decisions log v2.159). Base44 has **no CLI deploy-to-preview** and the editor
> Preview depends on the wedged GitHub auto-sync, so this is the reliable test-before-live path.
> It serves the **latest local commit** against the **LIVE backend** (real data + real auth, incl.
> email/password), via the Base44 vite plugin's `/api` proxy. **Nothing is deployed.**

**Recipe (laptop or desktop):**
1. `git pull --rebase` (gets committed **`.env.development`** with the public app params — already in repo).
2. `npm install` (only if deps changed).
3. `BASE44_LEGACY_SDK_IMPORTS=true ./node_modules/.bin/vite --port 5173 --host 0.0.0.0`
   (**always include `--host 0.0.0.0`** — owner standard 2026-06-26 — so another device on the SAME
   network can reach this machine's preview at `http://<this-machine-LAN-IP>:5173`, e.g. the desktop
   at `192.168.7.198`. LAN access needs both on the same network + the macOS firewall to allow `node`.)
4. Open **http://localhost:5173** (or the LAN IP from another device), log in normally. You'll be your
   own Base44 account. Look for the plugin log line
   **`[base44] Proxy enabled: /api -> https://app.serenovahub.com`** (confirms live data).
5. Stop with `pkill -f vite` (or Ctrl-C) when done.

**Why not `base44 dev`:** it runs a LOCAL backend that needs **Deno** AND the `base44/entities` move-aside
(which strips entity definitions so user/membership/event lists don't populate) and only wires Google
auth. Plain Vite + the live `/api` proxy avoids all of that. (The owner's `syncFromGithub` fn only READS
repo files via the GitHub REST API — it does NOT push code into the editor or rebuild.)

## 💻 SECOND-MACHINE (laptop) SETUP — do this before working on the laptop
1. **`git pull --rebase` BOTH repos.** This pulls the committed **`.env.production`** + the
   `src/api/base44Client.js` origin fallback — no manual env setup needed.
2. **`base44 login` ON THE LAPTOP (one-time per machine).** The CLI auth token is stored on disk and does
   **NOT** sync via git — each machine logs in separately. In a REAL terminal:
   `npx -y base44@0.0.56 login` → device-code → confirm at https://app.base44.com/login/device (act fast,
   it expires). Then `npx -y base44@0.0.56 whoami` should show `adamdcjones@gmail.com`. Token persists.
3. **Node/npm present** (laptop has been the build machine, so fine). `npx` fetches the CLI on demand —
   no global install needed; pin `base44@0.0.56`.
4. Now build + deploy exactly as the ⚙️ section above. App id `68c22e8ff3726c063c4a53e2`.
- **Preview testing:** use the **🔍 LOCAL PREVIEW** recipe above (plain Vite + live `/api` proxy) — the
  reliable test-before-live path. (The old `syncFromGitHub`/editor-Preview and `base44 dev` notes are
  superseded — see that section for why.)

---

## 📍 Live pointers (update on every push)

| What | Value |
|---|---|
| Code repo `main` HEAD | `18ffc17` — "Serenova → 0.7.0629: Guest List 'Assigned to' fixes + row redesign" |
| Serenova version | repo **0.7.0629** — **LIVE, deployed + verified** (bundle-grepped against `app.serenovahub.com`) |
| Docs | Decisions log **v2.184**, build phases **v2.79**, CLAUDE.md refreshed 2026-07-14T19:15 EDT |
| New tracked phase (not started) | **Phase TH — Global Theme Centralization.** `hubThemes.js` has correct teal/tan values, zero usages; system default is still blue `--harmony-*` in `Layout.jsx`. See decisions log 2026-07-14T18:00 EDT. |
| Memory repo HEAD | (this commit) — local clone `/Users/adamjones/Developer/perplexity_project_memory`. |
| Build | green |
| Base44 DEPLOYS | ✅ Site deployed 3× this session (`0.7.0627`/`28`/`29`), each verified live via bundle grep. No new *entities* provisioned live yet — `AccountFinancialDefaults.admin_fee_names` is a schema-only change, untested against a cold Base44 schema cache (watch for silent failures if new fee names stop persisting). |
| Pending live-verify | Nothing outstanding — all three deploys confirmed live before session end. |

## 🟢 Active work (claim before building; clear when done)

> Table reset 2026-07-14 — all prior rows were weeks-old and already done; full history lives in the
> decisions log if needed. **Full detail on every row below is in `SERENOVA-SESSION-HANDOFF.md`'s TOP
> BRIEF "UNFINISHED" section — read that before starting, this table is just the index.**

| Machine | Task | Status | Notes |
|---|---|---|---|
| _(unclaimed — owner priority 1)_ | Guest List **Export dropdown redesign** — "Export All [PDF] [CSV]" split icon buttons instead of one full-row button | next | `src/components/events/edit/GuestListTab.jsx` (`exportMenu`/`downloadCsv`). |
| _(unclaimed — owner priority 2)_ | Guest List **clean titled PDF** (By Show / By Date) | next | No PDF exists yet for guest lists (CSV only). Reuse the server-side jsPDF pattern from `generateFinancialSummaryPdf`; mind the 50-new-function cap — repurpose an existing fn like the project's established convention. |
| _(unclaimed — owner priority 3)_ | **Public shareable Guest List link** (+ retrofit the expiry rule onto EXISTING itinerary share links) | next | Model on `ShareableLink`/`generateShareableLink`/`validateShareableLink`/`getSharedItineraryData`/`SharedEventView.jsx`/`MobileShareDialog.jsx`. Expiry = final performance date + 14 days, computed dynamically (not stored), skip for `link_type:'staged_session'`. |
| _(unclaimed — owner priority 4)_ | **SystemHub "Staged Users" browser UI** | next | Backend DONE (`adminStagedSessions` `list_all` action, deployed). Needs: clickable Overview KPI tile (`HubOverview.jsx`) → new 3rd Users sub-tab in `SystemHub.jsx` (mirror existing `usersSubTab` deep-link pattern) → filterable (All/Active/Inactive) table. Also worth revisiting: "All Users" requiring a 2-char search isn't actually more secure (backend loads everyone regardless) — just an undocumented UX choice; "System Users" shows its list with no search needed. |
| _(unclaimed — low priority, logged only)_ | Same `NumberInput` snap-to-zero bug in `TeamRateSettings.jsx`/`FinancialDefaults.jsx` (Settings page) | backlog | The fix (`src/components/financial/NumberInput.jsx`) already exists — just needs wiring into these two files, not done this session (different page, out of scope for the EventFinancialDetails pass). |

> When you pick up a task: put your machine name in the row, set Status to "in progress",
> commit+push this file, THEN start. When done, set "done <date>" and update the pointers table.

---

## Repos & locations
- **Code:** `jonesadamd/serenovahub_b44` (`main`) — desktop clone at
  `/Users/adamjones/Developer/serenovahub_b44` (laptop path may differ).
- **Memory:** `jonesadamd/perplexity_project_memory` (`main`) — this folder. Holds the handoff +
  this sync doc. The repo `/docs` folder (in the code repo) is the source of truth for decisions/
  architecture/build status; this memory repo is the cross-session quick-load + coordination layer.
