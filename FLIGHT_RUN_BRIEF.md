# Flight Run — Four Tasks
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Mode:** Sequential. Complete each fully before moving to next. Write results to files. No push.

---

## TASK 1 — ThunderBrowser TB-0-3 through TB-0-5

Read `/home/ubuntu/.openclaw/workspace/project_jon/THUNDERBROWSER_PHASE01_TICKETS.md` for full ticket specs.

**TB-0-3 — platform.ts shim + webextension-polyfill**
Create `/home/ubuntu/thundergate-dev/extensions/thunderbrowser/lib/platform.js`:
- Thin shim: `export const runtime = chrome.runtime` etc. — wraps `chrome.*` so future Safari port can swap to `browser.*` without touching every call site
- Vendor `webextension-polyfill` as a single bundled file at `lib/vendor/browser-polyfill.js` (download from CDN or generate minimal stub if no npm available)
- Update `background/service-worker.js` and `content/content.js` to import from `lib/platform.js` instead of calling `chrome.*` directly

**TB-0-7 — Device keypair generation** (renumbered from TB-0-7 per ticket doc)
Add to `options/options.js`:
- On first load, check `keypair` store via `lib/storage.js`
- If no keypair: generate Ed25519 keypair via `crypto.subtle.generateKey({name:"Ed25519"}, false, ["sign"])` — non-extractable private key
- Export public key as JWK, store `{id: "device", publicKeyJwk, createdAt}` in IndexedDB `keypair` store
- Display public key fingerprint (first 16 chars of base64) in options page UI

**TB-0-4 — Alarm-driven SW heartbeat**
Add to `background/service-worker.js`:
- On SW install/activate: create `chrome.alarms.create("tb.keepalive", {periodInMinutes: 0.4})` (25s interval)
- On alarm: log "ThunderBrowser SW heartbeat" + check WSS state (stub for now — just log "WSS: not connected" since TB-0-5 isn't done yet)
- Export `getSwBootTs()` — timestamp when SW last started (used by popup)

**TB-0-5 — WSS client (stub connecting to mock)**
Add to `background/service-worker.js`:
- WSS client that connects to `ws://localhost:9876/browser` (mock TG endpoint — TB-0-10 will be the real mock)
- On connect: log "ThunderBrowser WSS connected"
- On disconnect: log "ThunderBrowser WSS disconnected", schedule reconnect in 5s
- On message: log "ThunderBrowser WSS message: " + data
- Expose `isConnected()` for popup to read
- Update popup to show real connected/disconnected state from SW

Build must be clean after each ticket. Run `node --check` on all JS files.

Write completion notes to `/tmp/task1-thunderbrowser-tb03-05.txt`

---

## TASK 2 — Build 28b Review vs Brief

Read both:
- `/home/ubuntu/.openclaw/workspace/build28b-ios-patch.md` (794 lines)
- `/tmp/cli-jon-context/BUILD28_PRESSURE_TEST_BRIEF.md` (just updated — 410 lines, all 8 bugs)

Compare. Identify:
1. What's in build28b that is NOT in the pressure test brief
2. Any wire-protocol corrections in build28b that need to be in the brief
3. Any new bugs in build28b that should be added to Build 28 scope

The brief already has a Section K flagging some build28b items. Check if that section is complete or if anything was missed.

Write findings to `/tmp/task2-build28b-review.txt`
If anything critical is missing from the brief, update `/tmp/cli-jon-context/BUILD28_PRESSURE_TEST_BRIEF.md` to add it.

---

## TASK 3 — ThunderGate Rebuild + Clean Restart

All of tonight's code changes need to be cleanly deployed:

1. `cd /home/ubuntu/thundergate-dev && npm run build 2>&1` — must be clean
2. Check what's running: `pgrep -a -f "thundergate-dev/dist" | head -5`
3. Kill old process if running: `kill <pid>`
4. Start fresh: `sudo systemctl start thundergate`
5. Verify: `thundergate ghost status`
6. Run learn-test to confirm learning loop is still passing: `node dist/cli/main.js ghost learn-test 2>&1`
7. Check the 8-check Doctor green report: `thundergate ghost status` — all 8 checks visible

Write status to `/tmp/task3-thundergate-restart.txt`

---

## TASK 4 — aa_reserve.py v1.8 Full Patch

Write the complete patched script.

Read `/home/ubuntu/.openclaw/workspace/scripts/aa_jumpseat_reserve.py` as the reference (structural twin of aa_reserve.py on Rex).

Also read the handler module already written: `/home/ubuntu/.openclaw/workspace/scripts/aa_remind_later_handler.py`

Write a complete `aa_reserve_v1.8.py` to `/home/ubuntu/.openclaw/workspace/scripts/aa_reserve_v1.8.py` that:
- Is a full working script (not just a diff)
- Integrates the "Remind Later" handler at the correct point in the login flow
- Adds detection for the password expiry interstitial: URL pattern `/password/change` OR text matching `/your password (has |is )?expired/i` OR button text "Remind Me Later" / "Remind Later"
- On detection: clicks "Remind Later" / "Remind Me Later" button and continues
- If click fails or button not found: logs clearly and raises an exception (don't silently continue past password expiry)
- Version comment at top: `# aa_reserve.py v1.8 — adds Remind Later handler`

Write build notes to `/tmp/task4-aa-reserve-v18.txt`

---

## Completion

Write summary to `/tmp/flight-run-complete.txt`
Update `/tmp/cli-jon-context/ACTIVE_TASKS.md`

## Rules
1. Read cli-jon-context first
2. Write files directly — no terminal output  
3. No push
4. Sequential — finish each before starting next
5. npm run build clean after Task 1 and Task 3
