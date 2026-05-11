# Finish Run — Outstanding Items from Burn Run
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Complete everything the burn run didn't finish
**Mode:** WRITE CODE. Take as long as you need. No push.

---

## Context

The burn run (gentle-lobster) got killed. TB-1-1 content script is done and build is clean.
Still missing:
1. ThunderGate browser bridge
2. APNs iOS spec for Mack
3. TB-1-2 navigate action
4. TB-1-3 click/fill actions
5. Build 28b brief APNs sections

Read everything first: cli-jon-context repo, THUNDERBROWSER_EXTENSION_DESIGN.md, THUNDERBROWSER_PHASE01_TICKETS.md, APNS_DESIGN_BRIEF.md, BUILD28_PRESSURE_TEST_BRIEF.md.

---

## TASK 1 — ThunderGate Browser Bridge

Create `/home/ubuntu/thundergate-dev/src/browser/bridge.ts`:

A WebSocket server that:
- Listens on port 9877 (separate from main gateway 8765)
- Accepts ThunderBrowser extension connections (the `thunderbrowser.v1` subprotocol)
- Handles the hello/ready handshake from the design doc (§2.2)
- Maintains per-extension session state: `{ extensionPairId, devicePubKeyJwk, connected, currentRunId, lastSeen }`
- Routes commands from ThunderGate → extension and results back
- Receives and stores audit records to context.db (stub for now — just log them)
- Issues stub scope tokens (signed JWT with hardcoded permissions for dev use)
- Handles heartbeat pings
- Graceful reconnect: queues up to 30s of commands when extension disconnects, drains on reconnect

Wire into ThunderGate startup: import and start the browser bridge alongside the main gateway. Add to `src/core/runtime.ts` or wherever ThunderGate initializes its services.

`npm run build` must be clean after.

---

## TASK 2 — APNs iOS Spec

Write `/tmp/cli-jon-context/APNS_IOS_SPEC.md` — complete iOS implementation spec for Mack.

Read `/home/ubuntu/.openclaw/workspace/scripts/apns_server.py` (server side done).
Read `APNS_DESIGN_BRIEF.md` in this repo for full spec.

Cover:
- Xcode setup (Push Notifications entitlement, Background Modes → Remote notifications)
- AppDelegate changes needed
- First-launch permission prompt — full Swift code:
  ```swift
  UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, error in
      if granted {
          DispatchQueue.main.async { UIApplication.shared.registerForRemoteNotifications() }
      } else {
          // Show in-app banner: "Miss nothing when the app is closed"
          // Deep link to Settings → ThunderCommo → Notifications
      }
  }
  ```
- Device token registration (POST to ThunderBase /api/apns/register)
- Notification handling — open to correct channel on tap
- Bug #9 fix — outbound queue on reconnect: only retry unacked messages, clear acked on reconnect, send afterTimestamp
- What Mack needs from Apple Developer portal: APNs Auth Key (p8), Team ID, Key ID, Bundle ID
- Security: p8 key never committed to git, stored at `~/.thundergate/apns_auth.p8` on ThunderBase

---

## TASK 3 — TB-1-2 Navigate Action

The content script exists (TB-1-1 done). Now add the navigate action.

In the background service worker (`extensions/thunderbrowser/background/service-worker.js`):
- Handle `navigate` command from ThunderGate bridge
- Use `chrome.tabs.update({url})` for existing tab or `chrome.tabs.create({url})` for new tab
- Wait for load: listen for `chrome.tabs.onUpdated` with `status="complete"`
- Verify URL is in domain allowlist before navigating
- Return `{ tab_id, final_url, load_ms }`
- Handle `wait_for_load` command: wait up to timeout_ms for tab to finish loading

---

## TASK 4 — TB-1-3 Click + Fill Actions

Add to content script files:
- `dom-write.js` — new file in `/extensions/thunderbrowser/content/`
- Handle `click(ref)` command: find element by ref handle from the registry, dispatch MouseEvent click
- Handle `fill(ref, value, secret)` command: set input value, dispatch input + change events; if secret=true replace value with [REDACTED] in any snapshot
- Handle `scroll_to(ref)` command: element.scrollIntoView()
- Handle `press_key(key, modifiers)` command: dispatch KeyboardEvent
- Register all actions on the message bus

Update content.js to import dom-write.js (add to manifest content_scripts array too).

---

## TASK 5 — Update Build 28b Brief

Add to `/tmp/cli-jon-context/BUILD28_PRESSURE_TEST_BRIEF.md`:

**Section L — APNs Integration Tests:**
- Device token registration: app launches → registers → POST to ThunderBase → token stored
- Push delivery: ThunderBase sends push → iOS receives → notification appears
- Notification tap: tap notification → app opens → navigates to correct channel
- Background delivery: app killed → push arrives → app wakes → message visible

**Section M — Bug #9 Replay Fix:**
- Network switch test: switch WiFi→cellular mid-session → reconnect → verify NO message dump
- App background test: background app for 30s → foreground → verify only new messages delivered
- afterTimestamp test: reconnect sends correct afterTimestamp → relay only delivers missed messages
- Queue behavior: unacked messages retry on reconnect, acked messages do NOT resend

**Section N — First Launch Notification Prompt:**
- Fresh install: first launch → permission prompt appears immediately
- Grant path: user grants → registers for remote notifications → token sent to ThunderBase
- Deny path: user denies → in-app banner appears with Settings deep link

---

## Completion

- `npm run build` clean
- Commit all thundergate-dev changes: `cd /home/ubuntu/thundergate-dev && git add -A && git commit -m "ThunderBrowser browser bridge + TB-1-2/1-3 navigate/click/fill"`
- Write completion to `/tmp/finish-run-complete.txt`
- Update ACTIVE_TASKS.md

## Rules
1. Read everything first — don't start coding until you understand the full picture
2. Write files directly, no terminal output
3. Build must stay clean after every TypeScript change
4. Take as long as you need — no time pressure
5. Commit at end
