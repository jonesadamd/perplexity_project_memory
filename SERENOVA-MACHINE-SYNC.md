# Serenova Hub — Machine Sync & Coordination

> **Read this right after `SERENOVA-SESSION-HANDOFF.md`.** Two machines (desktop + laptop) each
> run an AI coding session against the same code repo (`jonesadamd/serenovahub_b44`) and share
> this memory repo (`jonesadamd/perplexity_project_memory`) via git. This doc is the lightweight
> protocol that keeps the two sessions from clobbering each other or duplicating work.
> Last Updated: 2026-06-21T11:30 EDT

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
| Code repo `main` HEAD | (Push Group 2 + approver-notify) — "Push Group 2: approver notify on request" |
| Serenova version | **0.6.0529** |
| Memory repo HEAD | (this commit) |
| Build | green |
| Pending live-verify | **Push Group 2** at `v0.6.0529` (deploy → `Notification` entity + updated `decideGuestRequest` + `submitStagedGuestRequest` provision → approve/deny pushes requester; new request notifies the event TM / fallback artist+AMC) · also **B4.1** at `v0.6.0525` (Push Group 1 ✅ verified) |

## 🟢 Active work (claim before building; clear when done)

| Machine | Task | Status | Notes |
|---|---|---|---|
| desktop | Handoff + sync-doc refresh | done 2026-06-21 | this update |
| laptop | Push + Travel-alerts planning doc | **done 2026-06-21** | `docs/PUSH-TRAVEL-ALERTS-WORKING-DOC.md` created; owner decisions locked (alerts=delays+gate only; day-of summary; on-entry lookup; **scheduler=GH Actions cron**, no native Base44 scheduler; tiered cadence). Build-phases→2.59, decisions-log→2.98. Planning only, nothing built. |
| _(unclaimed)_ | B4.2 — Verify-gated phone/email change | next | re-key Memberships on email change; Verify code on phone change |
| _(unclaimed)_ | Phase SA C part 2 — Expenses (`StagedRequests`) | queued | holding queue + receipt quarantine |
| laptop | Push+Travel-alerts **Group 1** (PWA Web Push delivery) | **✅ VERIFIED LIVE** (2026-06-21) | `0.6.0527`. Staged band/crew member enabled + got the self-test push end-to-end. Covers logged-in AND staged (identity via session token). `npm:web-push` works under Base44 Deno — risk cleared. |
| laptop | Push+Travel-alerts **Group 2** (guest push: decision + request approver-notify) | **BUILT — verify pending** (2026-06-21) | `0.6.0529`. `decideGuestRequest` pushes the requester on decision; `submitStagedGuestRequest` notifies the approver(s) on a NEW request — **event Tour Manager only, fallback artist/owner + AMC confirm-role members** (owner rule). Email + push + `Notification` row. No new secret (VAPID reused). ⚠️ deploy → 2 fns + entity provision. Follow-up: logged-in (non-staged) request path is a client write → no server notify (staged is dominant). **`dispatchNotification` dispatcher + prefs deferred to Group 3**. |
| _(unclaimed)_ | Push+Travel-alerts **Group 3** (GH-Actions cron + flight delay/gate alerts + on-entry lookup; builds `dispatchNotification`) | next | needs `NOTIFY_INTERNAL_SECRET` + a secret-gated `asServiceRole` entry; tiered cadence; idempotent alert dispatch |

> When you pick up a task: put your machine name in the row, set Status to "in progress",
> commit+push this file, THEN start. When done, set "done <date>" and update the pointers table.

---

## Repos & locations
- **Code:** `jonesadamd/serenovahub_b44` (`main`) — desktop clone at
  `/Users/adamjones/Developer/serenovahub_b44` (laptop path may differ).
- **Memory:** `jonesadamd/perplexity_project_memory` (`main`) — this folder. Holds the handoff +
  this sync doc. The repo `/docs` folder (in the code repo) is the source of truth for decisions/
  architecture/build status; this memory repo is the cross-session quick-load + coordination layer.
