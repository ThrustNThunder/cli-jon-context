# ThunderBrowser — Phase 0 Complete (TB-0-6 through TB-0-11)
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Complete all remaining Phase 0 tickets
**Mode:** WRITE CODE. 30 minutes. Go hard. No push.

---

## Context

TB-0-1 through TB-0-5 + TB-0-7 are DONE. Extension scaffold exists at:
`/home/ubuntu/thundergate-dev/extensions/thunderbrowser/`

Read `/home/ubuntu/.openclaw/workspace/project_jon/THUNDERBROWSER_PHASE01_TICKETS.md` for full specs.

---

## Remaining Phase 0 Tickets

### TB-0-3 — Already Done (platform.js exists)
Skip — verify `lib/platform.js` exists, move on.

### TB-0-6 — Popup status: real connected state
The popup currently hardcodes `connected: false`. Now that TB-0-5 WSS client exists:
- Popup polls SW via `chrome.runtime.sendMessage({type:"tb.status"})` every 2s
- SW responds with `{connected: bool, runId: string|null, scopeLabel: string|null, swBootTs: number}`
- Popup renders: green pill "Connected" or red pill "Disconnected", current task label if running

### TB-0-8 — Pairing flow
Options page pairing flow:
- "Pair with ThunderGate" button
- On click: generate QR code displaying `{ extensionId, pubKeyJwk fingerprint, pairingCode: random 6-digit }` as JSON encoded in QR
- Use a simple QR library (qrcodejs CDN or inline minimal implementation)
- Display QR + pairing code as text fallback
- Poll for pairing confirmation: `chrome.runtime.sendMessage({type:"tb.pairing.status"})` every 2s
- On confirmed: show "Paired ✅" + ThunderGate endpoint + device key fingerprint
- Store pairing state in IndexedDB `pairing` store (from TB-0-2)

### TB-0-9 — Dev launcher script
Create `/home/ubuntu/thundergate-dev/extensions/thunderbrowser/dev-chrome.sh`:
```bash
#!/bin/bash
# ThunderBrowser dev launcher — opens a fresh Chrome profile with the extension loaded
CHROME_PROFILE="/tmp/thunderbrowser-dev-profile"
EXTENSION_DIR="$(cd "$(dirname "$0")" && pwd)"
mkdir -p "$CHROME_PROFILE"
google-chrome \
  --user-data-dir="$CHROME_PROFILE" \
  --load-extension="$EXTENSION_DIR" \
  --no-first-run \
  --no-default-browser-check \
  "$@" &
echo "ThunderBrowser dev Chrome launched (profile: $CHROME_PROFILE)"
```
Make executable: `chmod +x dev-chrome.sh`
Also create a README.md in the extension root explaining how to use it.

### TB-0-10 — Mock ThunderGate server
Create `/home/ubuntu/thundergate-dev/extensions/thunderbrowser/mock-tg/mock-server.js`:
- Simple Node.js WebSocket server on port 9876
- Handles the ThunderBrowser WSS protocol (from design doc §2.2)
- On connect: sends `hello` event
- Accepts `ready` event, logs it
- Exposes a simple REPL: type commands to send to connected extension
- Built-in test scenarios:
  - `scenario navigate` → sends navigate command to extension
  - `scenario snapshot` → sends snapshot_dom command
  - `scenario click` → sends click command (dummy ref)
  - `scenario scope` → mints a fake scope token and sends it
- Logs all events from extension with timestamps

Create `/home/ubuntu/thundergate-dev/extensions/thunderbrowser/mock-tg/package.json` with `{"type":"module"}` and a start script.

### TB-0-11 — Fixture site (AA portal states)
Create `/home/ubuntu/thundergate-dev/extensions/thunderbrowser/fixture-site/`:
- Simple Express server (`server.js`) on port 7860
- Static HTML pages for each AA portal state:
  - `index.html` → login page (email + password form)
  - `dashboard.html` → AA dashboard with Travel nav
  - `travel-planner.html` → Travel Planner empty state
  - `results.html` → Search results (3 fake flight options)
  - `confirm.html` → Confirm trip page with the "Confirm trip" button (precision click target)
  - `confirmed.html` → Confirmed reservation page
  - `password-expired.html` → Password expiry interstitial with "Remind Later" button
  - `captcha.html` → CAPTCHA blocked page
  - `timeout.html` → Session timeout modal
- Each page has a consistent nav showing current state
- `package.json` with start script: `node server.js`

---

## After All Tickets

1. `npm run build` in thundergate-dev root — must be clean
2. Write completion to `/tmp/tb-phase0-complete.txt` with file listing
3. Update `/tmp/cli-jon-context/ACTIVE_TASKS.md`

---

## Rules
1. Read cli-jon-context + THUNDERBROWSER_PHASE01_TICKETS.md first
2. Write files directly — no terminal output
3. No push
4. GO HARD — 30 minutes, get as far as possible
5. If you finish all tickets, start TB-1-1 (content script DOM reader) as bonus
