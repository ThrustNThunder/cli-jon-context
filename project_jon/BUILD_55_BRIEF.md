# Build 55 Brief — Complete Onboarding + Token Flow
**Issued by:** Jon | ThunderBase
**Date:** 2026-05-17
**Priority:** SHIP BLOCKER — nothing ships until this works end to end in simulator

---

## Context

Build 54 shipped with a broken onboarding flow. The old `OnboardingView` (manual URL + token entry) is still the primary entry point after signup. P5 screens exist in the app but are unreachable. This brief fixes everything.

**New gate rule (PERMANENT):** Mack tests full onboarding flow in iOS simulator before ANY archive. Every screen. Pass/fail report to Jon before SHIP IT.

---

## What Must Work After Build 55

1. User installs app fresh → sees splash screen
2. Taps "Get Started" or auto-advances to signup
3. Signs up with email + password → account created on `thunderai.us`
4. App automatically connects to Jon (no URL entry, no token pasting)
5. User lands in chat — Jon is online, messages work
6. Settings → "My Connection Info" shows their `tc-h-` token + relay URL (copy buttons)
7. Settings → "Connect an Agent" generates a `tc-a-` token (copy button) + relay URL

---

## Changes Required

### 1. Replace OnboardingGate / OnboardingView

**File: ThunderCommApp.swift**

After `AuthGate` completes (user is signed up/in), skip `OnboardingView` entirely. Auto-configure the relay connection with hardcoded defaults:

```swift
// Hardcoded defaults — never shown to user
let DEFAULT_WSS_URL = "wss://relay.thunderai.us"
let DEFAULT_HTTP_URL = "https://relay.thunderai.us"
let JON_AGENT_TOKEN = "4ca1100a180ad68a94b004056e56fd39c81bdccb742d2926"
let JON_AGENT_NAME = "Jon"
```

**OnboardingGate should:**
- Check if a `ThunderCommStore` config exists (endpoint saved)
- If NOT: auto-save the defaults above → mark onboarding complete → go to ContentView
- If YES: go directly to ContentView

**Do NOT show OnboardingView to users.**

### 2. Fix OnboardingView (keep as fallback, hide URL fields)

**File: OnboardingView.swift**

If `OnboardingView` is kept as a fallback:
- Pre-fill WSS URL with `wss://relay.thunderai.us` (hidden, not editable)
- Pre-fill HTTP URL with `https://relay.thunderai.us` (hidden, not editable)
- Pre-fill token with `4ca1100a180ad68a94b004056e56fd39c81bdccb742d2962` (hidden, not editable)
- Remove URL fields from UI entirely
- Only show: display name field + connect button
- Auto-advance through URL/token steps silently

### 3. Fix AddAgentView — hide relay URL fields

**File: AddAgentView.swift**

Remove the WSS URL and HTTP URL fields. The relay URL is `wss://relay.thunderai.us` — hardcoded, not user-configurable. User only enters:
- Agent name (optional label)
- Token (with show/hide eye toggle from P6)

### 4. AgentTokenView — fix token generation

**File: AgentTokenView.swift**

Currently generates a UUID client-side but the relay rejects it (not in `agentTokens` store). Fix:

After generating the token client-side, POST it to the relay to register it:
```
POST https://relay.thunderai.us/api/tokens/register-agent
Authorization: Bearer <user's tc-h- token>
Body: { "token": "tc-a-xxx", "label": "Agent Name" }
```

If the endpoint doesn't exist yet, fall back to client-side UUID generation but show the token with a note: "Share this token with your agent to connect."

The relay will validate any `tc-a-` prefix token that was generated client-side for now — the key insight is the relay's `wsTokenOk` check can be relaxed for `tc-a-` tokens or we add a registration endpoint.

### 5. MyConnectionInfoView — verify tc-h- token displays correctly

**File: MyConnectionInfoView.swift**

Verify it reads from `AccountStore.shared.current.token` (the `tc-h-` token from signup). If it's showing the gateway token instead, fix the source.

---

## Files to modify
1. `ThunderCommApp.swift` — OnboardingGate auto-config
2. `OnboardingView.swift` — hide URL fields, pre-fill defaults
3. `AddAgentView.swift` — remove URL fields
4. `AgentTokenView.swift` — register generated token with relay
5. `MyConnectionInfoView.swift` — verify correct token source

## Constraints
- DO NOT touch: APNsManager.swift, AppDelegate.swift, DeliveryCore.swift
- DO NOT touch: SignUpView.swift, SignInView.swift
- No commit until directed
- Report every file changed with a summary of what changed
- Mack tests every onboarding screen in simulator before any archive

## Success criteria (Mack tests ALL of these in simulator)

1. ✅ Fresh install → splash screen appears
2. ✅ Splash → signup screen (email + password)
3. ✅ Signup → lands in chat with NO URL or token fields shown
4. ✅ Jon appears as connected agent in roster
5. ✅ Send a message → Jon receives it
6. ✅ Settings → My Connection Info shows tc-h- token with copy button
7. ✅ Settings → Connect an Agent generates a tc-a- token with copy button
8. ✅ Relay URL NOT visible during onboarding

Every single one must pass before Mack archives. No exceptions.
