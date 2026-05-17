# Build 54 Onboarding Fix Brief
**Issued by:** Jon | ThunderBase
**Date:** 2026-05-17
**Priority:** CRITICAL — Build 54 cannot ship to external users without this

---

## The Problem

`ThunderCommApp.swift` has this structure:
```
SplashGate → AuthGate → OnboardingGate → ContentView
```

- `AuthGate` shows `SignUpView` / `SignInView` — email/password auth ✅ works
- `OnboardingGate` shows `OnboardingView` — OLD manual token entry screen ❌ wrong

The old `OnboardingView` asks users to manually enter:
- Gateway WSS URL
- HTTP URL  
- Bearer token
- Display name
- Then runs a connection test

This is the pre-P5 flow. It blocks every new user. The P5 screens exist in the app but are unreachable.

---

## The Fix

Replace `OnboardingGate` / `OnboardingView` with a simplified flow that:

1. **After signup completes** (user has `tc-h-` token from `AuthGate`) → automatically configure the relay connection using hardcoded defaults
2. **Skip the manual URL/token entry entirely** — the app already knows the relay URL and Jon's token
3. **Go straight to ContentView**

### Hardcoded values (never shown to user):
- WSS URL: `wss://relay.thunderai.us`
- HTTP URL: `https://relay.thunderai.us`  
- Jon's agent token: `4ca1100a180ad68a94b004056e56fd39c81bdccb742d2926`
- Jon's display name: `Jon`

### What to do

**Option A (simplest):** Replace `OnboardingGate` body so that if the user has a valid auth session (from `AuthGate`), it auto-configures and goes straight to content:

```swift
struct OnboardingGate<Content: View>: View {
    // ... existing state
    var body: some View {
        if isOnboarded {
            content
        } else {
            // Auto-onboard with defaults instead of showing OnboardingView
            Color.clear.task { await autoOnboard() }
        }
    }
    
    func autoOnboard() async {
        // Create a default ThunderCommStore config using hardcoded relay values
        // Save to UserDefaults so isOnboarded becomes true
        // Then flip isOnboarded = true
    }
}
```

**Option B (also fine):** Keep `OnboardingView` but pre-fill all fields with the hardcoded values and hide them. Make step 1 (gateway) auto-advance. User only sees name entry + notifications permission.

**Recommended: Option A.** Cleanest. The relay URL should never be user-visible during onboarding per the P5 brief.

### What isOnboarded checks

Find where `OnboardingGate` checks if onboarding is complete. It's likely checking `ThunderCommStore` for a saved endpoint/token. Make the auto-onboard write those same values so the check passes.

---

## Also fix: Relay URLs visible in AddAgentView

`AddAgentView` shows `wsURL` and `httpURL` fields. These should be hidden — the relay URL is hardcoded and not user-configurable.

Hide or remove the URL fields from `AddAgentView`. Just show:
- Agent name field
- Token field (with show/hide toggle per P6)
- Connect button

The relay URL is always `wss://relay.thunderai.us` — no user input needed.

---

## Files to modify
- `ThunderCommApp.swift` — replace OnboardingGate logic OR
- `OnboardingView.swift` — if keeping the view, pre-fill and hide URL fields
- `AddAgentView.swift` — hide relay URL fields

## Constraints
- Do NOT touch: APNsManager.swift, AppDelegate.swift, DeliveryCore.swift
- Do NOT touch: SignUpView.swift, SignInView.swift (auth works fine)
- Do not commit until directed
- node --check not applicable (Swift) — just report what changed
- Report every file changed

---

## Success criteria

After this fix:
1. User installs app fresh
2. Taps through signup (email + password)  
3. Goes directly to chat UI — no URL fields, no token pasting
4. Jon is already connected as the default agent
5. Messages work

That's the experience Alex and Burt need.
