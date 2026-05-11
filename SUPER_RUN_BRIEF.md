# Super Run — All Five Tasks
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Mode:** Sequential. Complete each task fully before moving to the next. Write results to files. No push.

---

## TASK 1 — PBS Reader Dry Run (TIME SENSITIVE)
PBS bidding closes May 13 @ 1200 CT. Rex (Dell, Tailscale 100.114.29.10) is currently offline.

**What to do:**
1. Check if Rex tunnel is reachable: `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 openclaw gateway call ping --url ws://100.114.29.10:18789 --token a0ae5d6f36f51f7ef3ccff4048c60a1e37828dde1536cb4f --json 2>&1`
2. If Rex is online: send him a message asking him to run `python3 ~/.openclaw/workspace/scripts/pbs_reader.py` and report the output. Use: `openclaw gateway call chat.send --url ws://localhost:18793 --token a0ae5d6f36f51f7ef3ccff4048c60a1e37828dde1536cb4f --params '{"sessionKey":"agent:main:main","message":"Rex - urgent. PBS bidding closes May 13. Please run python3 ~/.openclaw/workspace/scripts/pbs_reader.py and report back what you see. Michael needs to review bids before close.","idempotencyKey":"pbs-check-0511"}' --json`
3. If Rex is offline: document his status and flag clearly that PBS manual intervention may be needed before May 13.

Write results to `/tmp/task1-pbs-status.txt`

---

## TASK 2 — Build 28 Pressure Test Brief
Write a complete CLI Jon pressure test brief for ThunderCommo Build 28 iOS.

Build 28 changes (bridge side already live on ThunderBase):
- Ack-on-receipt fix: bridge acks immediately on WS receipt, dispatches async — kills 12s timeout bug
- DM routing fix: replies tagged with correct channel (direct vs tnt)
- Tap-to-retry: watchdog timer clears on ack receipt
- afterTimestamp replay fix: iOS sends afterTimestamp on reconnect

The brief should tell CLI Jon exactly what to test on the iOS side, what pass/fail looks like for each fix, and what the pressure test command sequence is.

Write to `/tmp/cli-jon-context/BUILD28_PRESSURE_TEST_BRIEF.md`

---

## TASK 3 — ThunderBrowser Phase 0 Scaffold
Start TB-0-1 and TB-0-2 from the build tickets.

Read `/home/ubuntu/.openclaw/workspace/project_jon/THUNDERBROWSER_PHASE01_TICKETS.md` for the full ticket specs.

**TB-0-1:** Create the Chrome MV3 extension scaffold at `/home/ubuntu/thundergate-dev/extensions/thunderbrowser/`:
- manifest.json (MV3, correct permissions, no host_permissions yet)
- background/service-worker.js (stub — just logs "ThunderBrowser SW started")
- content/content.js (stub — just logs "ThunderBrowser content script loaded")
- popup/popup.html + popup/popup.js (status pill: Connected/Disconnected)
- options/options.html + options/options.js (placeholder)
- icons/ (placeholder 16/32/48/128 — can be simple colored squares, SVG or PNG)

**TB-0-2:** Create the IndexedDB schema + storage layer at `/home/ubuntu/thundergate-dev/extensions/thunderbrowser/lib/storage.js`:
- All object stores from the ticket spec (keypair, pairing, scope, runs, audit, state_pack, settings)
- Typed get/put/delete/scan helpers
- Schema versioning (version 1)

Write build notes to `/tmp/task3-thunderbrowser-scaffold.txt`

---

## TASK 4 — aa_reserve.py v1.8 — Remind Later Handler
Read the current aa_reserve.py on Rex. Since Rex may be offline, read the local copy if available.

Check: `/home/ubuntu/.openclaw/workspace/scripts/aa_reserve.py`

The v1.7 script failed on May 10 because it hit a password expiry interstitial ("Remind Later" button) and had no handler. Michael manually saved the reservation.

Design and document the fix:
- What does the "Remind Later" interstitial look like? (button text, page pattern)
- What should the script do when it detects it? (click "Remind Later", continue)
- What's the selector/detection pattern?
- Write the patched handler code as a diff or standalone function

Write to `/tmp/task4-aa-reserve-patch.txt` and if the local script exists, write the patched version to `/home/ubuntu/.openclaw/workspace/scripts/aa_reserve_v1.8.py`

---

## TASK 5 — Ghost Jon Doctor Green 8-Check Definition
Implement the full 8-check Doctor green definition in ThunderGate.

From the pressure test analysis (`GHOST_JON_PRESSURE_TEST_ANALYSIS.md` in this repo), Doctor green requires ALL eight:
1. Weighted day score ≥ 0.75
2. Error rate < 0.05
3. Samples ≥ 10
4. `[ghost: not yet ready]` rate < 0.02
5. Zero FK errors newer than last deploy
6. Zero JSONL parse failures newer than last deploy
7. Learning-loop tests T1+T2+T3 still passing on deployed build (re-run nightly)
8. Ghost harness uptime ≥ 23h of previous 24

Read `/home/ubuntu/thundergate-dev/src/ghost/evaluator.ts` — current Doctor green only checks items 1-3.

Implement items 4-8 in the evaluator. Update `GhostDailyScore` interface to include the new fields. Update `thundergate ghost status` output to show all 8 checks.

Write build notes to `/tmp/task5-doctor-green.txt`

---

## Completion

When all five tasks are done, write a summary to `/tmp/super-run-complete.txt`:
- Task 1: Rex status + PBS situation
- Task 2: Build 28 brief location
- Task 3: ThunderBrowser scaffold location + file list
- Task 4: aa_reserve patch status
- Task 5: Doctor green implementation status

Update `/tmp/cli-jon-context/ACTIVE_TASKS.md`

---

## Rules
1. Read cli-jon-context (WHO_YOU_ARE, ARCHITECTURE) first
2. Write files directly — no terminal output
3. No push
4. Sequential — finish each task before starting the next
5. Be honest about blockers (Rex offline = flag it, don't fake a PBS result)
