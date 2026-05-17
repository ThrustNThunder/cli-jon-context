# Build 54 Badge Brief — Badge Count Behavior

**Date:** 2026-05-17  
**Author:** Jon  
**Status:** READY FOR CLI JON

---

## What Michael Wants (Non-negotiable)

- Badge increments for every new message that arrives while the app is in the background
- Badge clears to zero the moment Michael opens the app (foreground)
- That's it. No other badge behavior.

---

## Current State (after CLI Jon's last session)

**APNsManager.swift `handleSilentPush`:**
- Badge increment was removed. Silent push now only drains + calls `completion(.newData)`. ❌

**ThunderCommApp.swift:**
- `clearBadge()` call replaced with inline `UNUserNotificationCenter.current().setBadgeCount(0)` in `.onChange(of: scenePhase)` when `phase == .active`. ✅

**Net result:** Badge never increments. Clears on foreground. Badge is permanently zero. Wrong.

---

## Correct Implementation

### Design decision: increment in `DeliveryCore.drainInbox()`, not in `APNsManager`

`drainInbox()` already knows exactly how many new messages arrived (the `collected` array). That's the canonical count. `APNsManager.handleSilentPush` calls `drainInbox()` — so the increment happens naturally when the drain resolves. Don't duplicate the count logic in `APNsManager`.

### Changes required

#### 1. `DeliveryCore.swift` — add badge increment at end of `drainInbox()`

After the existing drain logic (ack, update lastDrainAt), add:

```swift
// Increment badge by the number of new messages received
let newCount = collected.filter { !seenIdsSet.contains($0.id) }.count
// NOTE: seenIds are marked BEFORE this check, so re-filter on what was actually new
// Better: track new message count during the loop
```

**Correct approach — track during the dedup loop:**

In `drainInbox()`, replace the existing dedup loop:
```swift
// BEFORE:
for m in collected where !seenIdsSet.contains(m.id) {
    inbound.append(m)
    markSeen(m.id)
}
```

With:
```swift
// AFTER:
var newMessageCount = 0
for m in collected where !seenIdsSet.contains(m.id) {
    inbound.append(m)
    markSeen(m.id)
    newMessageCount += 1
}
```

Then after `updateLastDrainAt(...)`, add badge increment **only if backgrounded**:

```swift
if newMessageCount > 0 && !sceneIsActive {
    await incrementBadge(by: newMessageCount)
}
```

Add private method to `DeliveryCore`:

```swift
private func incrementBadge(by count: Int) async {
    let center = UNUserNotificationCenter.current()
    let current = await center.badgeCount
    try? await center.setBadgeCount(current + count)
}
```

Add `import UserNotifications` to `DeliveryCore.swift` if not already present.

#### 2. `ThunderCommApp.swift` — badge clear on foreground (ALREADY CORRECT)

The current `.onChange(of: scenePhase)` block with `UNUserNotificationCenter.current().setBadgeCount(0)` when `phase == .active` is correct. **Leave it as-is.**

#### 3. `APNsManager.swift` — no changes needed

`handleSilentPush` already calls `DeliveryCore.shared.drainInbox()`. The badge increment now lives in `drainInbox()`. Nothing to change here.

---

## Files to touch

| File | Change |
|------|--------|
| `DeliveryCore.swift` | Add `newMessageCount` counter in drain loop, `incrementBadge()` method, `import UserNotifications` |
| `ThunderCommApp.swift` | No change needed — badge clear already wired correctly |
| `APNsManager.swift` | No change needed |

---

## Edge cases

- **App is foreground when drain runs:** `sceneIsActive` is `true` → badge does NOT increment. Messages are visible on screen. Correct.
- **Multiple pushes while backgrounded:** Each silent push triggers `drainInbox()`, each call increments by the new messages found. Badge accumulates correctly.
- **Duplicate messages:** `seenIdsSet` dedup in the loop means only genuinely new messages count. Replayed drain = 0 new = 0 badge increment.
- **User opens app:** `scenePhase → .active` → `setBadgeCount(0)`. Clean.

---

## DO NOT

- Do not use `UIApplication.shared.applicationIconBadgeNumber` — deprecated in iOS 17, will generate warnings
- Do not put badge logic in `APNsManager` — `drainInbox()` is the single source of new message count
- Do not increment badge when `sceneIsActive == true`
- Do not add a UserDefaults badge counter — `UNUserNotificationCenter.badgeCount` is the source of truth

---

## Verification

After building:
1. Send a message from Jon while app is backgrounded → badge should show "1" on app icon
2. Send 3 more → badge should show "4"  
3. Open the app → badge clears to 0
4. With app in foreground, receive a message → no badge change (already visible)

---

## Notes

- `UNUserNotificationCenter.badgeCount` is available iOS 16+. ThunderCommo targets iOS 16+. No compatibility issue.
- This is the correct UNUserNotificationCenter API: `setBadgeCount(_ newBadgeCount: Int) async throws`
- Badge permission is included in `requestUserAuthorization()` options: `[.alert, .badge, .sound]` ✅
