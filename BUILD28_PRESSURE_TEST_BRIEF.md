# ThunderCommo Build 28 — CLI Jon Pressure Test Brief
**Date:** 2026-05-11
**From:** Jon (ThunderBase)
**To:** CLI Jon (final pressure pass before Mack builds and ships Build 28)
**Mode:** Logic + on-device pressure. No push.
**Source of truth:** `build28-ios-patch.md` (bugs 1–8) + `build28b-ios-patch.md` (bugs 9–14, follow-on) + `BUILD28_BACKGROUND_SPEC.md` (TNT logo watermark).

---

## Context

Build 28 fixes **8 observable, user-visible iOS defects** against the Build 27 baseline. The bridge side is already live on ThunderBase (`thundercomm-stable`, deployed): `lastDispatchChannel` shipped (`bridge.mjs` lines 88, 246, 528, 574), ack-on-receipt live, no further bridge changes in this build. This pressure pass is **iOS-only** — Mack's source on `repos/thundergate-sparse` is the artifact under test.

The earlier brief mistakenly scoped Build 28 to four bridge-coupled fixes. **That was incomplete.** The full Build 28 cut list per `build28-ios-patch.md` is:

| # | Bug | Severity |
|---|---|---|
| 1 | DM Routing — replies tagged `direct:<agent>` but iOS dumps into `#tnt` bucket | 🔴 BLOCKER |
| 2 | Tap-to-Retry false positive — watchdog flips to `.failed` after bridge ack | 🔴 BLOCKER |
| 3 | DM context window — DM thread resets on view re-entry | 🔴 BLOCKER |
| 4 | Initial avatar badge — "M"/"J" circle next to composer is noisy/redundant | 🟡 UX |
| 5 | Phone number field — no keypad / no live mask | 🟡 UX |
| 6 | Settings save bug — fields revert to blank after Save | 🟡 BLOCKER for onboarding |
| 7 | Double hash `# #tnt` — UI double-prefixes channel name | 🟡 UX |
| 8 | Thinking dots per-agent — Jon shows none, Mack stuck on | 🟡 UX |

Bugs 9–14 (font sizes, `afterTimestamp`, background persistence, network transitions, streaming, untagged `#tnt`) are documented separately in `build28b-ios-patch.md`. They are **out-of-scope for this pressure pass** but flagged in Section H below so Mack can sequence them into Build 28b without losing thread.

Also covered: TNT logo watermark spec (Section I) — static visual layer behind the message list, ship-this for Build 28.

Carry-forward from Build 27: connectionEpoch (A1–A6), `localPeerId` alignment (B1–B3), feature set (Section C). None of those should regress.

---

# Wire-protocol facts (load-bearing for bugs 1, 3, 8)

These are what the bridge **actually sends** (verified against `extensions/thundercomm/bridge.mjs` on ThunderBase). The iOS side must match these exactly.

| Bridge → iOS payload | `channel` value |
|---|---|
| Reply from Jon to a `#tnt` mention | `"tnt"` |
| Reply from Jon to a DM | `"direct:jon"` |
| Echoed Michael DM (sender side) | `"direct:michael"` |
| Echoed Michael DM (target side) | `"direct:<agentId>"` |
| Echoed Michael `#tnt` message | `"tnt"` |
| Thinking event | `{ "type": "thinking", "agentId": "<id>" }` (no per-agent "stop" — ends when a `message` arrives for the same `agentId`) |

Consequences for pressure testing:

1. The wire `channel` is a **String**, not an enum. The legacy `ChannelType` (.team/.direct) cannot represent `direct:<agentId>` thread identity. If iOS still decodes `channel` as enum, BUG #1 will not be fully fixed.
2. A DM thread is uniquely keyed by the string after `direct:`. `direct:jon` is Michael's "DM Jon" thread; `direct:michael` is Jon's "DM Michael" thread.
3. There is no "stop thinking" event. Per-agent dots clear when a `message` lands for the same `agentId`.

---

# Pressure Test Tasks

## A. BUG #1 — DM Routing (BLOCKER)

**Bridge side (already live):** `bridge.mjs` tags replies with `channel: "direct:<agent>"` or `"tnt"` via `lastDispatchChannel`. Verify with the DM-Jon probe in the command sequence; the relay frame's `channel` field must equal `"direct:jon"`.

**iOS side — source-level verification:**

1. **`Models/Message.swift` (or wherever `ConversationMessage` is defined):**
   `channel` MUST decode as `String`, not `ChannelType`. The enum dropping `:jon` is the root cause. Look for `let channel: String` and a `dmAgentId` computed property (`channel.dropFirst("direct:".count)` extraction).
2. **`ThunderCommStore.swift`:** A single `messages: [ConversationMessage]` array is a FAIL. Required shape is `@Published private(set) var messagesByChannel: [String: [ConversationMessage]]` plus `@Published var activeChannel: String = "tnt"` plus `var visibleMessages: [ConversationMessage] { messagesByChannel[activeChannel] ?? [] }`.
3. **Inbound dispatcher (`case .message`):** Must call `handleInboundMessage(_:)` which buckets by `msg.channel` (defaulting empty string → `"tnt"`), enforces id-idempotency, and calls `clearThinking(forAgent: msg.agentId)`. The old `messages.append + currentlyThinking.remove` pattern is a FAIL.
4. **Outbound echo (`sendText`):** Optimistic insert must go through `handleInboundMessage(optimistic)` with the full wire channel string (`"tnt"` or `"direct:<id>"`), NOT a separate side-array.
5. **Channel-row taps in `ContentView.swift`:** Each row must set `store.activeChannel = "tnt"` or `"direct:\(agent.id)"`. Bind the list to `store.visibleMessages`.
6. **No fallback** that routes unknown channels into `tnt`. Malformed / missing channel = log-and-drop, NOT silent merge.

**Pass/fail:**
- ✅ PASS: DM-Jon thread shows ALL and ONLY DM-Jon traffic in order. `#tnt` shows zero DM-Jon messages. Switching DM-Jon → `#tnt` → DM-Jon shows the full DM thread on both visits. Mention `@jon` in `#tnt` → reply lands in `#tnt`, not DM.
- ❌ FAIL: Any DM reply appears in `#tnt`. Any `#tnt` message appears in DM-Jon. DM thread resets to empty on view re-entry (overlap with BUG #3 — the same `messagesByChannel` change fixes both).

**On-device pressure sequence:**
```
1. From #tnt, send "@jon ping".
2. From DM-Jon (open Jon's row), send "ping".
3. From #tnt, send "@mack ping".
4. From DM-Jon, send "another DM".
5. Switch to #tnt — verify only the two #tnt messages + replies visible.
6. Switch to DM-Jon — verify all DM-Jon messages + Jon's replies visible, in order.
7. Background app 60s, foreground. Re-open DM-Jon. Full thread still present.
```
Expected: zero cross-contamination at any point.

---

## B. BUG #2 — Tap-to-Retry False Positive (BLOCKER)

**Bridge side (already live):** ack fires synchronously on WS receipt before any dispatch. Tail-verify: `journalctl -u thundercomm-bridge -n 200 | grep -E "(ack-receipt|dispatch)"` shows ack BEFORE dispatch for the same `messageId`.

**iOS side — source-level verification:**

1. **Per-message watchdog tokens** in `ThunderCommStore.swift`: `sendWatchdogs: [String: Task<Void, Never>]` keyed by messageId (NOT a single global timer). The watchdog Task body must guard `if self.deliveryState[messageId] == .sending` before flipping to `.failed` — sticky `.sent`/`.delivered` must NEVER downgrade.
2. **idempotencyKey → messageId map:** `idempotencyKeyToMessageId: [String: String]` exists so the ack handler can find the right id.
3. **`case .ack` handler:** calls `markSent(messageId:)` which cancels the watchdog and sets `.sent` (or leaves `.delivered` intact). Single await context, no debouncing, no batching.
4. **`DeliveryCore` (if integrated):** `markFailed` refuses downgrade from `.sent`/`.delivered` — monotonic transition guard at ~line 57–68. If the actor pattern is in place, the local-state version in 2a of the patch is redundant; what matters is the ack handler calls `await delivery.markSent(messageId:)`.
5. **`MessageBubble` retry render:** Button visible ONLY when `deliveryState[message.id] == .failed`. NOT visible for `.sending`, `.sent`, `.delivered`. If predicate is `state != .delivered`, that's the bug.
6. **connectionEpoch gate:** ack handler reads epoch under the same MainActor as the bump. Stale-epoch acks dropped at WS client layer (Build 27 A2 — all 8 callback sites gated). A stale ack must NOT `markSent` the wrong message.

**Pass/fail:**
- ✅ PASS: send/ack/reply on a 200ms+ link never trips watchdog. Zero red overlays on acked messages. No `.failed` flash.
- ❌ FAIL: Any `.failed` state on a message that was acked within 12s. Any retry overlay rendered when ack actually arrived.

**On-device pressure sequence:**
```
1. Cold start, sign in.
2. Open #tnt, send 10 messages back-to-back at human cadence (~1/sec).
3. Send 10 more rapidly (~5/sec).
4. Background 30s, foreground, send 5 more.
5. Toggle airplane ON 8s, OFF, send 5 more.
6. Toggle airplane ON 15s, OFF, send 5 more.
```
Expected: every message ends `.delivered`. Messages sent **during** airplane-on MUST surface `.failed` with retry (correct — bridge never saw them). Messages that landed during the next reconnect window must NOT show retry. 12s watchdog constant unchanged unless Mack lowered it for Build 28 — document if lower.

---

## C. BUG #3 — DM Context Window (BLOCKER)

**Problem:** DM thread appears to reset / show only partial context when reopened. Same root cause as BUG #1 — messages not persisted per-channel, so view switches threw the thread away.

**iOS side — source-level verification:**

1. **`messagesByChannel` from BUG #1 is the fix.** DM messages live in `messagesByChannel["direct:jon"]` independent of `activeChannel`, so reopening the DM restores the full thread. If BUG #1's bucketing is in place and DM threads still reset, the regression is in step 2 or 3 below.
2. **History rehydration on connect (`case .history`):** Must iterate `history.messages` and route EACH through `handleInboundMessage(_:)` so they land in the right per-channel bucket. A line like `messages = history.messages + messages` is a FAIL — lumps everything into one array.
3. **Persistence layer (`thundercomm.messagesByChannel.v1`):** If Mack settled on UserDefaults / SQLite / Core Data, the storage key must store a **per-channel map**, NOT a single array. `restore()` called from `init()`, `persist()` debounced from `handleInboundMessage` (~250ms `Task.detached` is fine).
4. **`lastMessageId`:** still tracked off `history.messages.last` for resume-watermark continuity.

**Pass/fail:**
- ✅ PASS: 5-message DM exchange → switch to `#tnt` → switch back → full 5 messages visible. Cold-restart → DM thread still has all 5 messages (if persistence is wired). Bridge `history` on reconnect → messages land in right buckets.
- ❌ FAIL: DM thread resets to empty / partial on view re-entry. History rehydration dumps DM messages into `#tnt`.

**On-device pressure sequence:**
```
1. Open DM-Jon, exchange 5 messages.
2. Switch to #tnt, switch back. Full thread visible.
3. Background 60s, foreground. Full thread still visible.
4. Hard-quit (swipe from Recents), relaunch. DM thread restored.
5. Toggle airplane 20s → reconnect fires → no DM thread truncation, no rehydration leak into #tnt.
```

---

## D. BUG #4 — Initial Avatar Badge (UX)

**Problem:** "M" / "J" initial circles next to the composer input are noisy and redundant — the name label / channel header already conveys identity.

**iOS side — source-level verification:**

1. **`ContentView.swift` (or `ComposerBar.swift`):** the composer `HStack` must NOT contain `AvatarBadge(initial: identity.initial)`. The first element should be the `TextField`, not a badge.
2. If a similar avatar appears next to the recipient name in the DM header, that must also be gone — brief specifies the name label is sufficient.
3. **`AvatarBadge` view definition:** if no longer used anywhere, remove it. Do NOT leave a placeholder `Color.clear` — let the text field own the leading edge.

**Pass/fail:**
- ✅ PASS: `#tnt` composer = text field + send button only. No "M" circle. DM-Jon composer = clean, no "J" circle next to input or in DM header.
- ❌ FAIL: Avatar badge anywhere in the composer chrome.

**On-device pressure sequence:**
```
1. Open #tnt → screenshot composer.
2. Open DM-Jon → screenshot composer + DM header.
3. Diff against Build 27 screenshots: badges gone, layout otherwise unchanged.
```

---

## E. BUG #5 — Phone Number Field Format (UX)

**Problem:** Settings phone field uses default QWERTY keyboard, no live mask.

**iOS side — source-level verification:**

1. **`SettingsView.swift`:** phone `TextField` carries `.keyboardType(.phonePad)` and `.textContentType(.telephoneNumber)`.
2. Persistence backing is `@AppStorage("user_phone") private var phoneRaw: String` (the raw 10-digit string — masked render is a separate `@State` mirror that derives from `phoneRaw`).
3. `.onChange(of: phoneDisplay)` filters to digits, prefixes 10, and calls `formatPhone(_:)` to produce `(xxx) xxx-xxxx`. Pasting an unformatted string normalizes.
4. `formatPhone` handles partial input correctly:
   - 0 digits → `""`
   - 1–3 → `"(123)"`
   - 4–6 → `"(123) 456"`
   - 7–10 → `"(123) 456-7890"`
5. `onAppear` rehydrates the masked display from `phoneRaw`.

**Pass/fail:**
- ✅ PASS: Tap field → numeric keypad. Type `5551234567` → field shows `(555) 123-4567` live as user types. Save, reopen settings → `(555) 123-4567` rendered. Persisted raw is `5551234567`.
- ❌ FAIL: QWERTY appears. Mask doesn't apply. Reopen shows raw digits or empty.

**On-device pressure sequence:**
```
1. Settings → Phone field → keypad appears.
2. Type 5551234567 one digit at a time → mask updates each keystroke.
3. Paste "555-123-4567" → field normalizes to (555) 123-4567.
4. Paste "+1 (555) 123-4567 ext 99" → field normalizes to (555) 123-4567 (extras dropped).
5. Save, close settings, reopen → field still shows (555) 123-4567.
```

---

## F. BUG #6 — Settings Save Bug (BLOCKER for onboarding)

**Problem:** User enters data in settings, taps Save, fields revert to blank. Persistence layer dropping writes.

**iOS side — source-level verification:**

1. **`SettingsView.swift`:** `TextField`s bind **directly** to `@AppStorage` (or a Keychain-backed `@Published`), NOT to a `@State` mirror that gets reset on Save. Look for these exact bindings:
   - `@AppStorage("user_display_name") private var displayName: String`
   - `@AppStorage("user_phone") private var phoneRaw: String`
   - `@AppStorage("gateway_url") private var gatewayURL: String`
   - `@StateObject private var secrets = SecretsStore.shared` (Keychain wrapper for the token)
2. **`SecretsStore`:** `gatewayToken: String` with `didSet { write(key:value:) }`. Reads/writes Keychain via `SecItem*` APIs (kSecClassGenericPassword). The token must NOT live in UserDefaults.
3. **Save handler:** no writes needed there — bindings already wrote on keystroke. Save only kicks `gateway.configure(url:token:)` + `gateway.connect()`.
4. **Things that MUST be gone:**
   - Any `@State private var <fieldName>` mirroring an `@AppStorage` binding.
   - Any `isEditing` toggle that nukes bindings on toggle-off.
   - Any "saved!" confirmation that writes to `@State` and then reads from `@State`.

**Pass/fail:**
- ✅ PASS: Enter display name + phone + gateway URL + token → Save → close → reopen → all four populated. Cold-restart → still populated. Token in Keychain (verify via Xcode device console: `gateway_token` key exists in Keychain, NOT in UserDefaults plist).
- ❌ FAIL: Any field reverts to blank. Token appears in UserDefaults plist (security regression).

**On-device pressure sequence:**
```
1. Fresh install. Open Settings.
2. Enter display name "Michael", phone, gateway URL, token. Save.
3. Close Settings, reopen → all four fields populated.
4. Hard-quit, relaunch → all four populated (UserDefaults + Keychain survive).
5. Uninstall + reinstall → Keychain may or may not survive (iOS policy). UserDefaults definitely cleared. Note behavior.
```

---

## G. BUG #7 — Double Hash `# #tnt` (UX)

**Problem:** Channel name renders as `# #tnt`. UI prefixes `#` to a string that already contains `#`.

**iOS side — source-level verification:**

1. **`ContentView.swift` (sidebar row, header, anywhere else channel names render):** the offender is one of `Text("#\(channel.name)")`, `Text("# " + channel.displayName)`, or `Label("#\(activeChannel)", ...)` where the channel id already contains `#`.
2. **Required helper:** `extension String { var asChannelDisplay: String { ... } }` that:
   - For `"direct:<id>"` → `id.prefix(1).uppercased() + id.dropFirst()` (e.g. `direct:jon` → `Jon`)
   - For everything else → strip any existing leading `#`, then prefix exactly one (e.g. `tnt` → `#tnt`, `#tnt` → `#tnt`, NOT `# #tnt`)
3. **Decision (per Jon's patch):** store the raw channel id WITHOUT `#`, prefix in the view exactly once via `.asChannelDisplay`. Audit `ContentView.swift` for any `Text("#")` literal followed by a channel string — replace each.
4. **Defensive strip:** the helper's `hasPrefix("#")` check means even if some upstream still adds a `#`, only one renders.

**Pass/fail:**
- ✅ PASS: Sidebar shows `#tnt`, header shows `#tnt`, DM rows show `Jon` / `Mack` (not `#direct:jon`).
- ❌ FAIL: `# #tnt` anywhere. `#direct:jon` anywhere. Empty/`#` in the channel switcher.

**On-device pressure sequence:**
```
1. Open app → sidebar = "#tnt" + "Jon" + "Mack". No "# #tnt".
2. Tap "#tnt" → header = "#tnt".
3. Tap "Jon" → header = "Jon" (or "DM Jon" — whatever the design spec says, but NOT "#direct:jon").
4. Grep deviceconsole / SwiftUI rendering for any "# #" sequence.
```

---

## H. BUG #8 — Thinking Dots Per-Agent (UX)

**Problem:** Jon shows no thinking indicator at all. Mack's indicator shows even when Mack isn't processing. Root cause: thinking state is global (single `Bool` or unscoped `Set<String>`), or keyed on the wrong agent.

**iOS side — source-level verification:**

1. **`ThunderCommStore.swift`:** `@Published var thinkingByAgent: [String: Bool]` keyed by `agentId`. Helpers: `setThinking(forAgent:)`, `clearThinking(forAgent:)`, `isThinking(_:)`. A single global `Bool` or unscoped `Set<String>` is a FAIL.
2. **`case .thinking` handler:** `setThinking(forAgent: thinkingMsg.agentId)`. The bridge emits `{ "type": "thinking", "agentId": "<id>" }` — no per-agent stop event.
3. **Stop-on-message:** `handleInboundMessage(_:)` from BUG #1 calls `clearThinking(forAgent: msg.agentId)`. That closes the loop without a separate wire event.
4. **Safety timeout:** `setThinking(forAgent:)` schedules a 60s cleanup `Task` so dots don't get stuck if the message never lands.
5. **Active-channel filter (`activeThinkingAgents`):**
   - In `#tnt`: any thinking agent.
   - In `direct:<id>`: only that agent.
   Computed off `store.thinkingByAgent.compactMap { $0.value ? $0.key : nil }`.
6. **`ThinkingDotsRow` rendering:** keyed by `agentId` (e.g. `.id("thinking-\(agentId)")`). One row per thinking agent, NOT a single global row.
7. **No hardcoded agent gating:** grep `ThunderCommStore.swift` and `ContentView.swift` for `agentId == "mack"` or `agentId != "jon"`. Either pattern means Jon was silently dropped. Remove. `thinkingByAgent[agentId]` is the ONLY thing that decides who shows dots.
8. **Roster sanity:** `store.roster` must contain `"jon"`. If `displayName(forAgent: "jon")` returns nil, the row is skipped — that's why Jon disappears.

**Pass/fail:**
- ✅ PASS: `@jon hi` in `#tnt` → "Jon is thinking…" appears, replaced by Jon's reply. DM-Jon "hi" → "Jon is thinking…" in DM only. DM-Mack while Jon replies in `#tnt` → DM view shows "Mack is thinking", `#tnt` view shows "Jon is thinking". 60s ceiling fires if reply never lands.
- ❌ FAIL: Jon never shows dots. Mack's dots stuck. Both agents share a single global row. Dots persist after reply lands.

**On-device pressure sequence:**
```
1. Send "@jon ping" in #tnt → Jon dots → Jon reply. Mack silent throughout.
2. Send "@mack ping" in #tnt → Mack dots → Mack reply. Jon silent throughout.
3. DM-Jon "hi" → Jon dots in DM only. Switch to #tnt mid-think → no Jon dots in #tnt.
4. While Jon's reply is in flight, send "@mack ping" in #tnt → BOTH agents show dots simultaneously in #tnt. Independent clear.
5. Force a 60s+ silence with no message → dots auto-clear (safety timeout).
```

---

## I. TNT Logo Background Watermark (visual ship-this)

Source: `BUILD28_BACKGROUND_SPEC.md` in `cli-jon-context`. **Scope for Build 28:** static watermark only. Dynamic pulse + user-selectable variants are deferred to Build 29-30.

**iOS side — source-level verification:**

1. **Background `ZStack` layer behind `messageListView`** in `ContentView.swift` (or wherever the chat root composes). Order: background → message list on top.
2. **Asset:** `Image("tnt-logo")` (the bolt/thunder mark). If asset is missing in the bundle, the placeholder is the `⚡` SF Symbol — Mack swaps in the real asset before TestFlight ships.
3. **Sizing:** `.resizable().scaledToFit().frame(width: UIScreen.main.bounds.width * 0.45)` — ~40-50% of the shorter screen dimension.
4. **Opacity:** `0.08` to `0.12`. Test both white and brand purple (#7B2FBE) tint on dark bg; ship whichever reads better. Build 28 is dark-mode only.
5. **Position:** vertically centered in the visible message-list area. Stays fixed — does NOT scroll with messages.
6. **Layout:** full-bleed (edge to edge). Behind the message list and the composer.

**Things NOT to ship in Build 28:**
- No `.animation(.easeInOut.repeatForever)` pulse (Build 29).
- No "App Background" settings panel (Build 30+).
- No web-UI version. iOS only. `bridge.mjs` / `web/app.js` untouched.

**Pass/fail:**
- ✅ PASS: Logo visible behind both `#tnt` and DM threads at 8-12% opacity, centered, ~45% of screen width. Doesn't scroll with messages. Doesn't compete with text legibility. Composer reads clean on top.
- ❌ FAIL: Logo scrolls with messages. Logo dominates (>15% opacity). Logo missing in DM view. Composer text unreadable due to overlay.

**On-device pressure sequence:**
```
1. Open #tnt → logo centered behind messages.
2. Scroll messages — logo stays fixed.
3. Switch to DM-Jon → logo still visible behind DM thread.
4. Side-by-side on iPhone 14 Pro and iPhone SE — sizing reads right on both.
5. Send a 5000-char message — composer + bubble legible against logo.
6. Verify dark mode only (no light-mode break — out-of-scope for Build 28).
```

---

## J. Regression check (Build 27 carry-forward, MUST NOT have broken)

| Item | Verify |
|------|--------|
| `connectionEpoch` is `@MainActor` or atomic | A4 from Build 27 — bare `var Int` = ship blocker |
| All 8 async callback sites epoch-gated | A2 — receive, send, ping, auth-timeout, URLSessionDelegate, outbound queue drain |
| `localPeerId` comparison uses `if let` guard | B1 from Build 27 |
| `localPeerId` NOT in `rowID(for:)` | B3 — would re-introduce BUG-7 streaming churn |
| `DeliveryCore` monotonic transitions intact | `markFailed` refuses downgrade from `.sent`/`.delivered` (line ~57–68) |
| `LightweightContextEngine` look-above still wired | DM short-circuit, depth-3 walk, confidence decay |
| Federation 45s idle terminate + 15s ping | `bridge.mjs` — must not have been touched |
| BUG-6 delete tombstone | Still honored on reload |
| BUG-10 Add Agent → bridge `register_agent` | Still functional |

---

## K. Out-of-scope (Build 28b — bugs 9-14)

These are documented in `build28b-ios-patch.md` and are NOT pressure-tested in this pass. Listed so Mack can sequence into Build 28b without losing context:

| # | Bug | Notes |
|---|---|---|
| 9 | Font sizes too large in `MessageBubble` | Body 14pt / sender 12pt / timestamp 11pt. Pure SwiftUI. |
| 10 | `afterTimestamp` on subscribe | Wire is **milliseconds since epoch**, not seconds. Brief got this wrong; trust the wire. Bridge accepts but currently returns empty history regardless. Ship the iOS side now. |
| 11 | Background connection persistence | `UIBackgroundTaskIdentifier` + exponential-backoff reconnect + ping/pong verify after foreground. |
| 12 | Network transitions (WiFi ↔ cellular) | `NWPathMonitor` + reconnect on interface switch. Don't replay on switch (watermark handles it). |
| 13 | Streaming responses | Wire `type: "stream"` (NOT `"delta"` — brief was wrong). Field carrying the token is `delta`. Per-agent streaming buffer. |
| 14 | Untagged `#tnt` response | Bridge-side only (`requireMention` gating on `dispatchToAgent`). NO iOS change. |

**Wire-protocol corrections** (already corrected in `build28b-ios-patch.md`, NOT in this brief's main body):
- Streaming event type is `"stream"`, not `"delta"`. Field is `delta` inside the `stream` envelope. Confirmed via `web/app.js:380`.
- `afterTimestamp` is **milliseconds since epoch**, matches `ConversationMessage.timestamp`. Omit the key on first-ever launch (don't send `null` or `0`).

---

## L. Build safety

- `node --check` passes on `bridge.mjs` (sanity, even though bridge is out of scope here).
- No force-unwraps on send / auth / identity / DM-routing paths.
- No `MainActor` violations in the ack callback or watchdog cancel paths.
- No retain cycles in the per-agent thinking-state observers.
- Xcode static analyzer clean on touched files.
- `messagesByChannel` Codable round-trip works (the persistence layer relies on it).
- Keychain wrapper handles `errSecItemNotFound` gracefully on first launch.

---

## Command sequence (CLI Jon's pressure run)

Run this verbatim from ThunderBase. Mack runs the iOS-side equivalent on his Mac with the app loaded on Michael's phone.

```bash
# 1. Bridge-side verification — already-live fixes
journalctl -u thundercomm-bridge -n 500 --since "1 hour ago" \
  | grep -E "(ack-receipt|dispatch|lastDispatchChannel|thinking|direct:)" \
  > /tmp/build28-bridge-trace.txt

# 2. Cold ping — confirm bridge is up
openclaw gateway call ping --url ws://localhost:18793 \
  --token a0ae5d6f36f51f7ef3ccff4048c60a1e37828dde1536cb4f --json

# 3. DM-Jon scoping probe — send a DM, watch the channel field
openclaw gateway call chat.send --url ws://localhost:18793 \
  --token a0ae5d6f36f51f7ef3ccff4048c60a1e37828dde1536cb4f \
  --params '{"sessionKey":"agent:main:direct:jon","message":"build28 dm probe","idempotencyKey":"b28-dm-1"}' --json

# 4. #tnt scoping probe — send a mention, watch the channel field
openclaw gateway call chat.send --url ws://localhost:18793 \
  --token a0ae5d6f36f51f7ef3ccff4048c60a1e37828dde1536cb4f \
  --params '{"sessionKey":"agent:main:main","message":"@jon build28 tnt probe","idempotencyKey":"b28-tnt-1"}' --json

# 5. Thinking-event observation — confirm per-agent agentId on the wire
openclaw gateway tail --url ws://localhost:18793 \
  --token a0ae5d6f36f51f7ef3ccff4048c60a1e37828dde1536cb4f \
  --filter 'type=thinking OR type=message' --json > /tmp/build28-thinking-trace.txt &

# 6. Latency floor — measure ack latency under load (50-message flood)
for i in $(seq 1 50); do
  openclaw gateway call chat.send --url ws://localhost:18793 \
    --token a0ae5d6f36f51f7ef3ccff4048c60a1e37828dde1536cb4f \
    --params '{"sessionKey":"agent:main:main","message":"b28 flood '"$i"'","idempotencyKey":"b28-flood-'"$i"'"}' --json &
done; wait

# 7. Watchdog probe — slow-reply DM (forces 12s window)
openclaw gateway call chat.send --url ws://localhost:18793 \
  --token a0ae5d6f36f51f7ef3ccff4048c60a1e37828dde1536cb4f \
  --params '{"sessionKey":"agent:main:direct:jon","message":"deliberate slow query","idempotencyKey":"b28-slow-1"}' --json
```

**Where to save artifacts:** `/home/ubuntu/thundergate-dev/THUNDERCOMMO_BUILD28_PRESSURE_REPORT.md` (mirror the Build 27 format).

---

## Output (CLI Jon's report)

Mirror `THUNDERCOMMO_BUILD27_PRESSURE_REPORT.md`:

- **Verdict:** **PASS** | **CONDITIONAL PASS** | **FAIL**
- **Per-bug verdict** (1 through 8) + TNT logo (Section I) + regression check (Section J)
- For each bug: source-confirmed status + on-device-confirmed status
- **Issues table:** **BLOCKER** | **WARNING** | **NOTE**
- If CONDITIONAL: exact 1–3 line code change Mack must apply before building
- If PASS: "Green light. Mack integrates Build 28 fixes, builds, ships to TestFlight."

**Verdict floor:** BUG #1, #2, #3, #6 are BLOCKERS. Any one of them failing = FAIL on the overall verdict, regardless of the rest. BUG #4, #5, #7, #8 and the TNT logo are degraders — CONDITIONAL PASS is acceptable if Mack can fix in <30 min and rebuild before TestFlight upload.

---

## Rules

1. Bridge changes are **NOT in scope** for this pass — already pressure-tested at the bridge level.
2. Mack's iOS source is the artifact. Reason from the implementation notes in `build28-ios-patch.md` if direct repo access is gated.
3. Be specific about file:line for any blocker.
4. If a fix passes at the source level but on-device behavior diverges, flag the divergence — don't paper over it.
5. The two wire-protocol corrections from `build28b-ios-patch.md` (streaming = `"stream"` not `"delta"`; `afterTimestamp` = ms) are out-of-scope for THIS pass but DO trust the wire over the brief on those if they bleed into Build 28 work.
6. No push.
7. Final verdict in the report. Michael reads the verdict line, not the whole doc — make it count.
