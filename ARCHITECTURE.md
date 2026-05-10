# Architecture — Key Decisions

## ThunderCommo iOS Delivery (LOCKED May 9)

**CORRECT model:**
- WebSocket = foreground-only optimization
- APNs = primary wakeup when backgrounded
- Server inbox = source of truth
- On every foreground: drain /api/inbox

**WRONG model (don't go back to this):**
- Keeping WebSocket alive in background
- iOS kills WS ~30s after backgrounding — this is Apple, not a bug

## Auth Model (LOCKED May 9)

- Email + password → server account
- Bearer token in Keychain (kSecAttrAccessibleAfterFirstUnlock)
- Face ID = local gate on Keychain entry, not server auth
- BYOAA: user adds their own agent (wss URL + token + KYA verify)
- KYA = Know Your Agent — display identity, future: keypair verification

## ThunderGate Runtime (Phase 1+2 complete)

22 design principles. Key ones:
- One context.db — all channels read/write same source
- TUI reads from session, doesn't own it
- Event-based learning (not time-based)
- Hybrid adaptive checkpoint (4K cold start)
- Ghost Jon shadow testing — 7 days Doctor green before cutover
- Doctor mode always running

## Model Routing

```
Routine ops → OpenClaw Jon (Anthropic API, pay per token)
Heavy work  → CLI Jon (Claude Max, flat rate)
```

CLI Jon reads this repo for context. No spawning needed for routine tasks.

## ThunderPhone (Future)

When Jon can make calls/texts for Michael:
- User identity verified via account + Face ID
- Agent authorized via KYA
- ThunderPhone checks authorization before any call
- The chain: Michael verified → authorizes Jon → Jon can act

This is WHY we built the auth system right from the start.

## BYOAA (Build 21+)

- User has their own ThunderGate instance
- Scans QR from their instance
- Agent imports cleanly
- Currently: Michael's gateway only
