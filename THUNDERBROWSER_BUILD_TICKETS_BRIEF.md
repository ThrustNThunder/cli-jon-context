# ThunderBrowser — Phase 0-1 Build Tickets Brief
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Convert ThunderBrowser extension design into concrete Phase 0-1 build tickets
**Mode:** Design + ticket writing only. NO CODE. Write output to file.

---

## Context

ThunderBrowser extension is fully designed — read `THUNDERBROWSER_EXTENSION_DESIGN.md` in this repo.

Michael confirmed: live viewable is the goal. Jon drives the browser, Michael watches his own Chrome screen, Jon narrates via ThunderCommo in real time. Phase 1 target = PBS bidding + AA automation.

**PBS bidding is a key use case:**
- PBS (pilot bidding system) has a stateful web UI
- Jon reads bid slots, fills preferences, walks the workflow
- Michael confirms the final "Save Ballot" click via ThunderCommo
- Same live DOM + real-time reaction model as AA automation

---

## What to Produce

Write concrete, actionable build tickets for Phase 0 and Phase 1 from the design doc.

**Phase 0 — Skeleton (target: 1 week)**
- Extension scaffold (MV3, manifest, icons, popup shell)
- Background SW + content script wire-up
- WSS client connect/disconnect/reconnect/heartbeat
- ThunderGate Browser Bridge (new `src/browser/bridge.ts`)
- Pairing flow (keypair generation, QR display, TG pairing record)
- Dev environment setup (separate Chrome profile, dev extension ID, fixture server scaffold)

**Phase 1 — Core Actions (target: week 2-3)**
- DOM snapshot (structured mode)
- query / get_text / get_url
- navigate + wait_for_load
- click + fill + scroll_to
- detect_modal + detect_error + detect_loading
- is_logged_in (cookie presence only, no values)
- Audit log (local IndexedDB chain)
- Domain allowlist enforcement
- Input redactor (passwords, CC fields)

**For each ticket include:**
- Title
- Description (what it does, why it matters)
- Acceptance criteria (how you know it's done)
- Dependencies (what must exist first)
- Estimated complexity (S/M/L)
- Owner hint (CLI Jon = ThunderBase side, Mack = Safari/iOS side)

**Also write:**
- PBS automation state machine spec (parallel to AA state machine in the design doc) — what are the PBS portal states, how does Jon walk them, what's the confirmation gate
- ThunderCommo live narration spec — what does Jon say to Michael via ThunderCommo while driving the browser? What events trigger what messages?

---

## Output

Write to `/tmp/cli-jon-context/THUNDERBROWSER_PHASE01_TICKETS.md`

Update ACTIVE_TASKS.md when done.

---

## Rules
1. Read THUNDERBROWSER_EXTENSION_DESIGN.md first — this is the source of truth
2. Design/tickets only — no code
3. No push
4. Be specific enough that a developer can pick up a ticket and start immediately
