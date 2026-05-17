# Build 54 P7 Brief — Roster Bug Fix + Clean Slate
**Issued by:** Jon | ThunderBase  
**Date:** 2026-05-17  
**Branch:** build34/apns-relay  
**Status:** READY FOR CLI JON (iOS part only)

---

## Context
P1-P6 and P8 are all committed on build34/apns-relay (tip: 79c1f34).
P7 has two parts:
- **P7a (iOS)** — roster bug fix in the who's online view. CLI Jon builds this.
- **P7b (server)** — backend clean slate reset. Jon handles this on ThunderBase directly. NOT CLI Jon's job.

---

## P7a — Roster Bug Fix (iOS — CLI Jon builds this)

### Bug
"Michael L" is showing as an online agent in the who's online / roster view. He is a human, not an agent. The roster needs to correctly separate humans (tc-h- tokens) and agents (tc-a- tokens).

### Fix

**1. Who's Online view — refresh on foreground**
The roster view must refresh every time the app comes to foreground, not just on initial connect.
- Find where the roster/who's-online view is rendered
- Ensure it calls a refresh on every `.active` scene phase transition
- The relay sends a fresh `roster` frame on every WS connect — make sure the view reacts to it

**2. Token-based role display**
When rendering roster entries:
- `tc-h-` prefix → show in "People" or "Humans" section
- `tc-a-` prefix → show in "Agents" section  
- Unknown prefix → show in "Agents" section (safe default)
- Never show the same token in both sections

**3. Stale roster entries**
If a roster entry has no recent activity and the WS connection is fresh, do not carry over stale entries from the previous session. On WS reconnect, clear the local roster and wait for the server's fresh `roster` frame before rendering.

### Files likely affected
- ContentView.swift (roster/who's online rendering)
- ThunderCommStore.swift (roster state management)
- ThunderCommModels.swift (RosterAgent model — may need a `role` field check)

### Constraints
- DO NOT touch: APNsManager.swift, AppDelegate.swift, DeliveryCore.swift
- DO NOT touch: OnboardingView.swift
- Do not commit until directed
- Report every file changed

---

## P7b — Backend Clean Slate (Jon handles on ThunderBase — NOT CLI Jon)

This is a server-side operation Jon runs manually after Build 54 is on TestFlight and installed on Michael's phone.

**Sequence (Jon runs in order):**
1. Clear all test user accounts from relay in-memory store (restart thundercomm-inbox.service)
2. Invalidate all existing tokens — clear tokens from relay memory
3. Clear APNs token store: `echo '{"tokens":[]}' > ~/.thundergate/apns_tokens.json`
4. Restart relay: `sudo systemctl restart thundercomm-relay`
5. Michael onboards fresh in Build 54 app — full signup flow (email, phone, password → tc-h-xxx token)
6. Jon generates fresh tc-a-xxx token for himself via relay API
7. Verify messages flow both ways
8. Verify roster shows Michael as human, Jon as agent
9. THEN send TestFlight invites to Alex + Burt

**This happens AFTER Build 54 is installed. Never reset backend while old app is the only install.**

---

## Deliverable (CLI Jon)
- List every file changed and what changed
- Working tree state
- Stop — do not commit until Jon directs
