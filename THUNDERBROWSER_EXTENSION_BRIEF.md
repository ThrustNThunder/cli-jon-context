# ThunderBrowser — Browser Extension Design Brief
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Design the ThunderBrowser Chrome/Safari extension — full architecture, no code yet
**Mode:** Design + architecture doc only. NO CODE. Write output to file.

---

## What We're Building

ThunderBrowser is a browser extension that gives ThunderGate/ThunderAgent live eyes and hands in the browser. Not screenshots. Not headless Playwright. A real extension running in Michael's actual browser, connected to ThunderGate via WebSocket.

**The vision:** "Hey Jon, go reserve my jumpseat on AA" → Jon opens Chrome, navigates AA portal using Michael's real logged-in session, fills the form, confirms the reservation, reports back. No bot detection. No CAPTCHA. No residential IP dependency. It's Michael's browser.

**Reference:** `project_jon/THUNDERBROWSER.md` — read the extension architecture section.

---

## What to Design

### 1. Extension Architecture

Design the full Chrome Manifest V3 extension:

- **Background service worker** — persistent WebSocket connection to ThunderGate. Receives commands, sends DOM snapshots back.
- **Content script** — injected into every page. Reads DOM, executes actions (click, fill, scroll, navigate). Reports live page state changes via message passing to background.
- **Popup UI** — simple status panel: connected/disconnected, current task, authorize/revoke button.
- **ThunderGate bridge** — the server-side component that translates Jon's intent ("click login button") into extension commands and receives DOM state back.

Design the message protocol between:
- ThunderGate ↔ Extension background worker (WebSocket)
- Background worker ↔ Content script (Chrome runtime messaging)

### 2. Capability Map

What can the extension do? Design the full action set:

**Navigation:** navigate to URL, go back/forward, wait for page load
**DOM read:** get full page text, get element by selector/text/role, get form state, get current URL + title
**DOM write:** click element, fill input, select dropdown, check checkbox, scroll to element, press key
**Page state:** detect popups/modals, detect errors (404, access denied, login expired), detect loading states
**Network:** (optional) intercept XHR/fetch responses for key API calls (AA specifically)
**Auth:** read cookies for a domain (to verify logged in), detect session expiry

### 3. BYOAA Integration

Every agent-initiated action = BYOAA-authorized. Design:
- How does the extension verify that Jon has authorization before executing an action?
- Authorization token format — what does the extension check?
- What happens if authorization expires mid-task?
- How is the action record written for court-admissibility?

### 4. AA Portal Specifics

The AA automation use case is the immediate target. Design how the extension handles:
- Login session detection (is Michael logged in to aa.com?)
- Travel Planner navigation (stateful, can't use direct URLs)
- The "Confirm trip" precision click
- Password expiry interstitials
- Popups and modals
- Session timeout / re-auth

### 5. Safari Web Extension

Chrome MV3 → Safari Web Extension conversion path:
- What's the Xcode wrapper process?
- What APIs differ between Chrome and Safari?
- Which capabilities degrade on Safari?
- This is Mack's lane — design the handoff point clearly

### 6. Security Model

This extension has massive capability — it can read and interact with ANY website in Michael's browser. Design the security boundaries:
- How does the extension authenticate to ThunderGate? (Token? Certificate?)
- How does it prevent a compromised ThunderGate from exfiltrating cookies/passwords?
- Allowlist of domains the extension will act on?
- Audit log of every action taken?

### 7. Development + Testing Path

How do you develop and test this without breaking Michael's real browser:
- Dev/prod extension IDs
- Test environment (separate Chrome profile)
- How to replay AA automation scenarios in test without hitting real AA servers

---

## Output

Write full design doc to `/tmp/cli-jon-context/THUNDERBROWSER_EXTENSION_DESIGN.md`

Sections:
1. Extension component map (diagram in text/ASCII)
2. Message protocol spec (ThunderGate ↔ Extension ↔ Content Script)
3. Action API (full list of commands + parameters + return types)
4. BYOAA authorization flow
5. AA portal automation design
6. Safari handoff spec for Mack
7. Security model
8. Development/testing path
9. What to build first (MVP scope)

Update ACTIVE_TASKS.md when done.

---

## Rules
1. Read cli-jon-context first (WHO_YOU_ARE, ARCHITECTURE, THUNDERBROWSER.md context)
2. Design only — no code
3. No push
4. Be specific — this doc becomes the build brief for when we have an extension developer
5. Write to file, no terminal output
