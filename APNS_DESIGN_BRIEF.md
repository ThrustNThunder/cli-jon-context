# ThunderCommo iOS — APNs Push Notifications Design
**Date:** May 11, 2026
**From:** Jon (ThunderBase) — Michael's direction in flight MIA→RDU
**Target:** Build 28b
**Mode:** Design + implementation. Write code. No push.

---

## Why This Matters

Michael used ThunderCommo from SLC hotel → airport → flight. Every time the app closed or backgrounded, messages piled up and replayed in a burst on reconnect. That's the problem APNs solves:

Message arrives → Apple pushes to phone → app wakes → message delivered immediately, even if app was closed.

APNs also fixes the replay/queue dump bug — see Bug #9 below.

---

## Bug #9 — Message Replay on Reconnect (Add to Build 28b)

**Root cause:** On reconnect, the outbound queue flushes ALL pending messages in a burst, including ones already sent and displayed.

**Fix:** On reconnect, clear outbound queue of anything already acked. Only retry genuinely unacked messages (those with no server ack received). Ties into the `afterTimestamp` fix in Build 28 — same root cause, different symptom.

Add this to the Build 28b pressure test brief as Bug #9.

---

## Deliverables

### 1. ThunderBase — APNs Server Side

Write `/home/ubuntu/.openclaw/workspace/scripts/apns_server.py` — a standalone APNs push sender:

**Token registration endpoint** (to be called by iOS app on launch):
- Store `{ device_token, user_id, app_bundle_id, platform: "ios", registered_at }` in a simple JSON file at `~/.thundergate/apns_tokens.json`
- Called by iOS app when it gets its APNs device token

**Push trigger logic:**
- Function `send_push(device_token, title, body, channel, message_id)` using Apple's APNs HTTP/2 API
- Auth: APNs Auth Key (p8 file) — store path in config, don't hardcode
- Endpoint: `https://api.push.apple.com/3/device/{device_token}` (production) or `https://api.sandbox.push.apple.com/3/device/{device_token}` (sandbox)
- Payload:
```json
{
  "aps": {
    "alert": { "title": "Jon", "body": "New message in #tnt" },
    "badge": 1,
    "sound": "default",
    "mutable-content": 1
  },
  "channel": "tnt",
  "messageId": "uuid"
}
```
- Use `httpx` or `requests` with HTTP/2 support (or `aioapns` library if available)
- JWT auth using p8 key: `{ "iss": team_id, "iat": now, "exp": now+3600 }` signed with ES256

**Integration with bridge.mjs:**
Document (don't implement yet — that's bridge code in Jon's lane) where the push trigger should fire:
- When a message is dispatched to a session and the relay detects the iOS client is disconnected
- How relay detects offline: no active WS connection for that device token

Write the server-side script + document the bridge integration point.

### 2. iOS Side — APNs Setup Spec (for Mack)

Write `/tmp/cli-jon-context/APNS_IOS_SPEC.md`:

**Xcode setup:**
- Enable Push Notifications capability in Xcode → Signing & Capabilities
- Enable Background Modes → Remote notifications
- `AppDelegate` changes needed

**First launch permission prompt:**
```swift
// On first app launch (check UserDefaults flag)
UNUserNotificationCenter.current().requestAuthorization(
    options: [.alert, .badge, .sound]
) { granted, error in
    if granted {
        DispatchQueue.main.async {
            UIApplication.shared.registerForRemoteNotifications()
        }
    } else {
        // Show in-app banner explaining why notifications matter
        // "Miss nothing when the app is closed"
        // Deep link to Settings → ThunderCommo → Notifications
    }
}
```

**Device token registration:**
```swift
func application(_ application: UIApplication, 
                 didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
    // POST to ThunderBase: /api/apns/register
    // { device_token: token, user_id: currentUser.id }
}
```

**Notification handling (app in background/killed):**
```swift
// In UNUserNotificationCenterDelegate
func userNotificationCenter(_ center: UNUserNotificationCenter,
    didReceive response: UNNotificationResponse,
    withCompletionHandler completionHandler: @escaping () -> Void) {
    // Extract channel from notification payload
    // Navigate to correct channel on app open
    completionHandler()
}
```

**Bug #9 fix — outbound queue on reconnect:**
```swift
// On WebSocket reconnect, only retry unacked messages
// Clear queue of messages where ack was received
// Send afterTimestamp so relay only delivers what was missed
```

**Apple Developer requirements (Mack must do these):**
1. Generate APNs Auth Key (p8) in Apple Developer portal → Certificates, IDs & Profiles → Keys
2. Note: Team ID, Key ID, Bundle ID
3. Send p8 file + Team ID + Key ID to Jon securely (never commit to git)
4. Jon stores in `~/.thundergate/apns_auth.p8` on ThunderBase

### 3. Update Build 28b Pressure Test Brief

Add to `/tmp/cli-jon-context/BUILD28_PRESSURE_TEST_BRIEF.md`:
- Section L: APNs integration tests
- Section M: Bug #9 replay-on-reconnect fix
- First launch notification permission prompt verification

---

## Rules
1. Read cli-jon-context first
2. Write files directly — no terminal output
3. No push
4. Update ACTIVE_TASKS.md when done
5. Write completion to `/tmp/apns-design-complete.txt`
