# Build 54 — Agent Onboarding Architecture Spec
Date: May 13 2026
Author: Jon
Status: SPEC — not yet implemented

## The Problem
Current ThunderCommo onboarding requires manual token entry or copy-paste setup. This is friction. The vision: one button push to connect phone to agent.

## The Architecture

### Agent Side (ThunderBase/ThunderMind)
The agent knows:
- Its own session IP address (e.g., relay.thunderai.us for remote, or local IP for LAN)
- Its gateway port (18789 by default)
- Its identity token (generated per session, scoped to this pairing)

The agent generates a pairing payload:
```json
{
  "agent_id": "agent:main:main",
  "relay_url": "wss://relay.thunderai.us",
  "session_token": "<short-lived pairing token, expires 5min>",
  "agent_name": "Jon",
  "agent_version": "thundergate-2026.5.13"
}
```

### Phone Side (ThunderCommo iOS)
Option A (ideal — zero friction):
- App opens → relay already knows what agents are registered to this account → shows list → tap to connect
- No token needed — relay acts as trust anchor for both sides

Option B (interim — minimal friction):
- Agent generates a short pairing code (6 alphanumeric chars, e.g. "JON-7X3")
- User types 6 chars in the app → resolved to full connection params server-side
- Much better than a 48-char token

Option C (current fallback):
- Full connection string copy-paste: wss://relay.thunderai.us?token=xxxx
- Still better than today's flow but has friction

### How to Get There (Build 54 Scope)
1. Add a /api/pairing endpoint to relay.mjs:
   - POST /api/pairing/generate → returns {code: "JON-7X3", expires_at: ...}
   - GET /api/pairing/resolve?code=JON-7X3 → returns full connection params
2. Add thundergate CLI command: thundergate pair → prints the pairing code
3. iOS: Add pairing code entry field to onboarding (6 chars, auto-submit)
4. When code resolves → app gets relay_url + session_token → connects automatically

### Future State (Option A)
Once user accounts are stable on relay.thunderai.us:
- Agent registers itself on account sign-in
- App authenticates with same account
- Relay introduces them automatically
- Zero friction, zero input from user

### Security
- Pairing codes expire in 5 minutes
- One-time use — code invalidated after first successful connection
- Session token is scoped to this device pairing only
- Full audit trail in provenance ledger

## Owners
- Jon: relay.mjs pairing endpoint, thundergate pair CLI command
- Mack: iOS pairing code entry UI, connection flow
- Gate: Jon's CLI Jon pressure test before any ship

## Biometric Vault Unlock (Build 54 scope)

ThunderGate now ships an encrypted PII vault (vault.db, AES-256-GCM,
PBKDF2 100k). Unlock is locked-by-default on every session start. For
Build 54, we extend the unlock surface to include biometric approval
from the paired iPhone so Jon never sees plaintext PII flow through the
chat surface to authorize an unlock.

Wire flow:
1. Jon needs a vault field → ThunderGate calls `VaultService.access(...)`,
   which throws `VaultLockedError`.
2. ThunderGate calls `VaultService.buildUnlockRequest(task, reason)` and
   relays the `vault_unlock_request` envelope through the active channel
   (today: ThunderCommo). The envelope carries:
   `{type, request_id, task, reason, requested_at, ttl_ms}`.
3. ThunderCommo iOS (Mack's lane): `LocalAuthentication` handler fires
   Face ID / Touch ID with the `task` and `reason` shown in the prompt.
4. On user approval, the device replies with a `vault_unlock_approval`
   envelope: `{type, request_id, approved, approved_at, device_id,
   device_signature, biometric_token}`. Approval drops on a 60s replay
   window.
5. ThunderGate receives the approval, verifies the `device_signature`
   against the paired-device key from the existing pairing flow, and
   calls `VaultService.unlock({source: 'biometric', biometricToken,
   ttlMs})`.
6. Vault stays unlocked for the requested TTL (default 30 minutes), then
   auto-locks. Every step writes a row to the provenance ledger keyed
   by the original task description.

Phase-1 stub (already shipped on the gateway): the gateway side accepts
a password alongside the approval token until the iOS keychain wrap
ships. The relay envelope is real and can be exercised end-to-end on
the live relay; the iOS side only needs to round-trip the approval
message and supply a non-empty `biometric_token` for Build 54.

See `src/vault/PROTOCOL_VAULT.md` in `thundergate-dev` for the
authoritative wire schema and field descriptions.

Owners:
- Jon: VaultService + protocol contract (shipped) + vault_unlock_approval
  consumer wired into the active channel routing.
- Mack: iOS LocalAuthentication handler — small change. Shows `task` +
  `reason` in the biometric prompt, signs `request_id || approved_at`
  with the existing device key, returns `vault_unlock_approval`.
- Gate: round-trip request → approval → unlock → access on a paired
  iPhone with vault.db populated.
