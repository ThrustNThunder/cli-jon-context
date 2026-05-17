# Inbox Persistence Brief
**Issued by:** Jon | ThunderBase
**Date:** 2026-05-17
**Priority:** CRITICAL — blocking Build 54 ship

---

## The Problem

The inbox (`run-inbox.mjs`) stores everything in in-memory Maps:
- `users` Map — accounts (email, password hash, id)
- `sessions` Map — tc-h- tokens → userId
- `inbox` Map — userId/token → messages[]

Every service restart wipes all data. This means:
- Accounts are lost on restart
- Messages are lost on restart  
- APNs token registrations are lost on restart
- Clean slate onboarding is pointless if inbox restarts again

## What To Fix

Add file-based JSON persistence for all three Maps. On startup, load from disk. On every write, flush to disk.

### Storage files (in `~/.thundergate/`)
- `inbox-users.json` — users Map
- `inbox-sessions.json` — sessions Map  
- `inbox-messages.json` — inbox Map (messages per key)

### Implementation pattern

For each Map, add:
1. `loadXxx()` — reads JSON file, populates Map on startup
2. `saveXxx()` — writes Map to JSON file (synchronous or async, either is fine)
3. Call `saveXxx()` after every write to that Map

### Specific locations to add saves

**users Map** — save after:
- `users.set(...)` in signup handler
- Any user update

**sessions Map** — save after:
- `sessions.set(...)` in signin handler
- `sessions.set(...)` in signup handler
- Session refresh

**inbox Map** — save after:
- `storeInbox(...)` calls (two places in POST /api/messages handler)
- After ack/delete operations

### Startup

At the top of `createInboxServer()` (or module-level), call:
```js
loadUsers();
loadSessions();
loadMessages();
```

### File format

Simple JSON. Example for users:
```json
{
  "user@email.com": { "id": "user-xxx", "email": "user@email.com", "passwordHash": "...", ... }
}
```

For inbox (Map<string, array>):
```json
{
  "user-xxx": [{ "id": "msg-1", "content": "hello", "timestamp": 123456 }],
  "tc-h-xxx": [...]
}
```

## Additional fixes needed (same session)

### 1. Inbox GET — resolve by userId
The GET /api/inbox handler already has a patch applied to look up by userId from session. Verify it's working correctly after the persistence fix. The chain is:
- App sends `Authorization: Bearer tc-h-xxx`
- Sessions map: `tc-h-xxx` → `{ userId: "user-xxx" }`
- Inbox map: `"user-xxx"` → messages[]

If the session lookup isn't working, check the `inboxSession` / `inboxKey` logic added earlier.

### 2. Bridge inbox store — correct recipient ID
The bridge stores messages with `to = INBOX_RECIPIENT_ID`. After persistence is added, this should be set dynamically. For now, the bridge should look up the registered device token's user_id from `apns_tokens.json` and use that as the recipient. The `pickPushTarget()` function already returns `{ deviceToken, userId }` — use that `userId` as the inbox recipient instead of the hardcoded constant.

Update `storeMessageInInbox()` in `bridge.mjs`:
```js
async function storeMessageInInbox(id, text, timestamp) {
  const target = pickPushTarget(); // already reads from apns_tokens.json
  if (!target) { console.warn('[Bridge] Inbox store skipped: no push target'); return; }
  try {
    const resp = await fetch(`${INBOX_URL}/api/messages`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${INBOX_AGENT_TOKEN}` },
      body: JSON.stringify({ id, to: target.userId, body: text, kind: 'text', timestamp: timestamp || Date.now(), sender: 'jon' }),
    });
    if (!resp.ok) console.warn(`[Bridge] Inbox store non-2xx: ${resp.status}`);
  } catch (err) { console.warn(`[Bridge] Inbox store failed: ${err.message}`); }
}
```

### 3. Remove INBOX_RECIPIENT_ID constant from bridge.mjs
It's no longer needed after fix #2. Delete the constant.

## Files to modify
- `/home/ubuntu/thundergate/extensions/thundercomm/run-inbox.mjs` — add persistence
- `/home/ubuntu/thundergate/extensions/thundercomm/bridge.mjs` — fix storeMessageInInbox to use dynamic userId

## Constraints
- Do not restart any running services — write files only
- Do not commit — Jon reviews and deploys
- Report every change made
- node --check both files when done
