# CLI Jon Brief — ThunderCommo Build 28
**Date:** 2026-05-11
**From:** Jon (ThunderBase)
**Target:** iOS source on Mack's Mac — ThunderCommIOS.xcodeproj

Read https://github.com/ThrustNThunder/cli-jon-context for context before starting.

---

## Mission

Fix 8 iOS bugs in ThunderCommo. These are **iOS-only bugs** — the bridge fixes for #1 and #2 are already live on ThunderBase. Your job is the Swift/iOS side only.

Do NOT push. Write files directly. Mack will integrate, build, and ship after Jon's pressure test.

---

## Bug List (Priority Order)

### 🔴 PRIORITY 1 — DM Routing (BLOCKER)
**Bug:** Direct messages to Jon work (ack confirmed), but Jon's reply comes back tagged `channel: "tnt"` instead of `channel: "direct"`. It leaks into team chat instead of staying in the DM thread.

**Bridge fix already applied (ThunderBase):** `lastDispatchChannel` tracking — bridge now sends reply with correct channel. iOS side needs to:
- Correctly scope incoming messages to the active DM thread when `channel` starts with `"direct"`
- NOT show DM replies in the #TNT feed
- DM conversation history must persist — full thread visible, doesn't reset on view

### 🔴 PRIORITY 2 — Tap-to-Retry False Positive
**Bug:** Every sent message shows red "tap to retry" even though messages are landing. iOS send watchdog was flipping messages to failed after 12 seconds.

**Bridge fix already applied:** ack now fires immediately on receipt. iOS side: verify the watchdog timer is cleared on ack receipt. If Mack already patched this locally in `ThunderCommStore.swift`, confirm it's clean and carry it forward.

### 🔴 PRIORITY 3 — DM Context Window
**Bug:** DM window should show full persistent conversation history. Currently appears to reset or only show partial context. User should see full thread when opening a DM, same as #TNT shows full history.

---

### 🟡 PRIORITY 4 — Initial Avatar Badge
**Bug:** Small "M" (Michael) and "J" (Jon) initials appear next to the text input box. Remove them — the name label is sufficient. Clean up the UI.

### 🟡 PRIORITY 5 — Phone Number Field Format
**Bug:** In the settings/options field where user enters a phone number, the keyboard doesn't switch to phone format and there's no format mask. Should use `keyboardType(.phonePad)` or `.numberPad` and format as phone number as user types.

### 🟡 PRIORITY 6 — Settings Save Bug
**Bug:** When user enters data in settings and hits Save, data does not persist — fields return to blank. Settings are not being saved to UserDefaults or Keychain. Fix persistence layer.

### 🟡 PRIORITY 7 — Double Hash in Channel Name
**Bug:** Channel name displays as `# #tnt` instead of `#tnt`. Extra `#` appearing — likely the UI prefixes `#` to a string that already contains `#`. Strip or fix the prefix logic.

### 🟡 PRIORITY 8 — Thinking Dots Per-Agent
**Bug (iOS specific):**
- Jon has NO thinking indicator at all
- Mack's thinking indicator shows even when Mack is NOT the one processing
- Should be: each agent shows their own dots ONLY when they are actively responding

Fix: thinking state must be keyed per-agentId. When a `thinking` event arrives with `agentId: "jon"`, only show dots on Jon's entry. When `agentId: "mack"`, only Mack's. Clear dots when the reply arrives for that specific agent.

---

## Files to Touch
- `ThunderCommIOS/ThunderCommStore.swift` — main state, message handling, DM routing, watchdog, thinking state
- `ThunderCommIOS/ContentView.swift` or equivalent — UI for channel names, avatar badges, thinking dots
- `ThunderCommIOS/SettingsView.swift` or equivalent — settings save/persist, phone number field
- Bridge files on ThunderBase are **NOT in scope** — Jon already patched those

## Rules
1. Read cli-jon-context first
2. Write files directly — no printing to terminal
3. No push
4. No changes to web UI files (bridge.mjs, style.css, app.js, index.html) — those are Jon's lane
5. Update ACTIVE_TASKS.md in cli-jon-context when done
6. Focus: DMs first, then work down the priority list
