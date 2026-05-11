# Burn Run — Everything Outstanding
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Mode:** FULL SEND. Write code. No limits. No push. Go until done.

---

## Context

TB Phase 0 (TB-0-6 through TB-0-11) is already running in a parallel session.
This run picks up everything ELSE that's outstanding.

Read cli-jon-context + all relevant docs before starting each task.

---

## TASK 1 — APNs iOS Spec (was cut short)

Write `/tmp/cli-jon-context/APNS_IOS_SPEC.md` — complete iOS implementation spec for Mack.

Read `/home/ubuntu/.openclaw/workspace/scripts/apns_server.py` (already written — server side done).
Read `APNS_DESIGN_BRIEF.md` in this repo for full spec.

Cover:
- Xcode setup (Push Notifications entitlement, Background Modes)
- First-launch permission prompt (full Swift code)
- Device token registration (POST to ThunderBase)
- Notification handling (open to correct channel on tap)
- Bug #9 fix: outbound queue on reconnect (only retry unacked, clear acked)
- What Mack needs from Apple Developer portal (p8 key, Team ID, Key ID)

---

## TASK 2 — ThunderGate Browser Bridge

Start the server-side browser bridge in ThunderGate.

Create `/home/ubuntu/thundergate-dev/src/browser/bridge.ts`:
- WebSocket server on port 9877 (separate from main gateway port 8765)
- Handles ThunderBrowser extension connections
- Per-extension session management
- Command queue + reply routing (ThunderGate → extension → ThunderGate)
- Audit record ingest (stores to context.db)
- Scope token issuance (stub for now — returns a signed JWT with hardcoded permissions)
- `npm run build` must be clean after

---

## TASK 3 — ThunderBrowser Phase 1 Start (TB-1-1 through TB-1-5)

Read the Phase 1 tickets in THUNDERBROWSER_PHASE01_TICKETS.md.

**TB-1-1:** Content script DOM reader
- Add declarative content script injection to manifest for `aa.com` and `localhost:7860` (fixture)
- Content script: implement `snapshot_dom(mode)` — returns structured DOM tree (interactive + landmark elements, inputs redacted)
- Implement `query(selector, text, role)` — returns matching elements with ref handles
- Implement `get_text(ref)` and `get_url()`
- Input redactor: blank `input[type=password]` and `input[autocomplete^="cc-"]`

**TB-1-2:** Navigate action
- Background SW: handle `navigate` command from ThunderGate bridge
- Use `chrome.tabs.update({url})` or `chrome.tabs.create({url})`
- Wait for load: listen for `chrome.tabs.onUpdated` with `status="complete"`
- Return `{tab_id, final_url, status_code}`

**TB-1-3:** Click + fill actions
- Content script: handle `click(ref)` — find element by ref handle, dispatch click event
- Content script: handle `fill(ref, value, secret)` — set input value, dispatch input/change events
- Secret values: never logged, replaced by `[REDACTED]` in any snapshot

---

## TASK 4 — Update Build 28b Brief with everything

Read current `/tmp/cli-jon-context/BUILD28_PRESSURE_TEST_BRIEF.md`.

Add:
- Section L: APNs integration tests (device token registration, push delivery, notification tap → correct channel)
- Section M: Bug #9 replay-on-reconnect fix (queue behavior on reconnect, afterTimestamp)
- Section N: First-launch notification permission prompt verification

---

## TASK 5 — Commit everything to git

```bash
cd /home/ubuntu/thundergate-dev
git add -A
git commit -m "ThunderBrowser Phase 0 complete + Phase 1 start + browser bridge stub"
```

Also commit workspace changes:
```bash
cd /home/ubuntu/.openclaw/workspace
git add -A  
git commit -m "APNs server, aa_reserve v1.8, ThunderBrowser docs"
```

---

## Completion

Write summary to `/tmp/burn-run-complete.txt`
Update ACTIVE_TASKS.md

## Rules
1. Go hard — this is the burn run
2. Write files directly
3. Build must stay clean (npm run build after each TypeScript change)
4. Commit at end (Task 5 explicitly)
5. No push to GitHub remotes
