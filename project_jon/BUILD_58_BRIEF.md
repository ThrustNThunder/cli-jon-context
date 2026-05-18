# Build 58 Brief — Complete Onboarding Wire-Up
**Issued by:** Jon | ThunderBase
**Date:** 2026-05-17
**Base branch:** build56/final
**Priority:** SHIP BLOCKER — all 7 items must pass simulator before archive

---

## The Core Problem

After signup, `AccountStore` is never seeded. `DeliveryCore` requires an `Account` object 
(wsURL + token) to connect to the relay. Without it — no relay connection, no roster, no messages.
Everything else is downstream of this one missing wire.

---

## Complete Correct Flow (IMPLEMENT THIS EXACTLY)

```
App Launch
├── AccountStore.hasAccount? (tc-h- token exists in Keychain)
│   ├── YES → Face ID prompt → Chat
│   └── NO → Signup screen (never Face ID on fresh install)

Signup screen
├── Email, password, phone (required, 10+ digits), display name
├── POST /api/auth/signup → tc-h- token returned
├── Store tc-h- in Keychain (AuthManager)
├── Seed AccountStore with: wsURL="wss://relay.thunderai.us", token=tc-h-
├── DeliveryCore.shared.handleScenePhase(.active) → connects to relay automatically
└── Navigate to → Your Token screen

Your Token screen
├── Shows tc-h- token (masked, show/hide toggle, copy button)
├── Banner: "Share this token with your agent in their chat window or TUI"
└── Next → APNs Primer screen

APNs Primer screen
├── Text: "Enable notifications so your agent can reach you"
├── "Enable" button → calls APNsManager.shared.requestUserAuthorization()
├── iOS system prompt fires → Allow/Deny
└── Next → Add Agent screen (skippable)

Add Agent screen
├── Agent name field (optional label)
├── "Generate Token" button → creates tc-a-<UUID> → copy button
├── Relay URL "wss://relay.thunderai.us" → copy button (read-only, hardcoded)
├── Footer: "Give both to your agent. They'll paste them into their gateway config to connect."
├── NO session ID field (remove it entirely)
├── On "Save": POST /api/tokens/register-agent with tc-h- bearer + tc-a- token
├── Save agent to ThunderCommStore with name, token, wsURL
├── Agent appears in roster as offline/pending immediately
└── "Skip" option → goes straight to Chat

Chat (blank slate)
├── Roster shows any saved agents (offline until they connect)
├── No hardcoded #tnt, #jmab, no Jon default
├── "Messages will appear here" empty state
└── When agent connects to relay → appears online in roster
```

---

## 7 Specific Code Changes

### 1. AuthGate — Fix Face ID timing
**File: ThunderCommApp.swift or AuthGate**

```swift
// CURRENT (wrong): fires Face ID even on fresh install
// CORRECT:
if AccountStore.shared.hasAccount {
    // show Face ID / biometric unlock
} else {
    // show SignUpView
}
```

Check `AccountStore.shared.current != nil` or `AuthManager.shared.peekToken() != nil`.
If nil → signup. Never Face ID on first install.

### 2. SignUpView — Seed AccountStore after signup
**File: SignUpView.swift, in `advanceFromProfile` after successful signUp call**

After `UserStore.shared.signUp(...)` succeeds and returns the `tc-h-` token:
```swift
// Seed AccountStore so DeliveryCore can connect
let account = Account(
    id: UUID().uuidString,
    wsURL: "wss://relay.thunderai.us",
    httpURL: "https://relay.thunderai.us", 
    token: tc_h_token,
    name: displayName
)
AccountStore.shared.add(account)
```

This is the single most important fix. Everything else depends on this.

### 3. OnboardingView — Add APNs primer step, remove session ID
**File: OnboardingView.swift**

Add a new step between "Your Token" and "Add Agent":
- APNs primer screen with "Enable Notifications" button
- Calls `APNsManager.shared.requestUserAuthorization()`

Remove from Add Agent step:
- The "Step 2 — Connect using their session ID" section entirely
- The `agentSessionID` TextField and everything referencing it

Add to Add Agent step:
- Relay URL display (read-only, hardcoded "wss://relay.thunderai.us") with copy button
- Footer: "Give both to your agent. They'll use these to connect."

### 4. OnboardingView — Save agent to ThunderCommStore when user hits Save
**File: OnboardingView.swift, in the Add Agent save action**

When user taps "Save" / "Connect":
```swift
// Register on server
await registerAgentToken(generatedAgentToken, label: agentName)

// Save locally so it appears in roster
let agent = RosterAgent(
    id: generatedAgentToken,
    name: agentName.isEmpty ? "Agent" : agentName,
    emoji: "🤖",
    model: nil,
    role: "agent",
    status: "offline"
)
// Add to DeliveryCore or ThunderCommStore roster
```

### 5. AddAgentView — Remove session ID, add relay URL, match onboarding form
**File: AddAgentView.swift**

Remove: Session ID TextField and its Section entirely.
Add: Relay URL row (read-only, "wss://relay.thunderai.us", copy button).
Footer: "Give this token and relay URL to your agent."
Make this form identical to the onboarding Add Agent screen.

### 6. SplashView — Bigger logo
**File: SplashView.swift**

Change logo display size from 260 to 340pt.

### 7. SettingsView — Remove hardcoded Jon as default agent
**File: SettingsView.swift**

Search for any reference to Jon, "Jon", "jon-agent", the gateway token, or any hardcoded agent entry. Remove it. Settings should show only agents the user has explicitly added.

---

## Files to touch
1. `ThunderCommApp.swift` — AuthGate fix
2. `SignUpView.swift` — AccountStore seeding (MOST IMPORTANT)
3. `OnboardingView.swift` — APNs primer, remove session ID, add relay URL, wire agent save
4. `AddAgentView.swift` — remove session ID, add relay URL, match onboarding form
5. `SplashView.swift` — logo 340pt
6. `SettingsView.swift` — remove hardcoded Jon agent

## DO NOT TOUCH
- DeliveryCore.swift
- APNsManager.swift  
- AppDelegate.swift
- AccountStore.swift (only read from it, don't rewrite)

---

## Gate — Mack tests ALL in simulator before archive

1. ✅ Fresh install → signup screen (NO Face ID)
2. ✅ Signup → goes to Your Token screen
3. ✅ APNs prompt fires during onboarding
4. ✅ Add Agent: token + relay URL visible, copy buttons work, NO session ID field
5. ✅ After onboarding → chat loads blank
6. ✅ Saved agent appears in roster as offline
7. ✅ Settings → no hardcoded Jon agent
8. ✅ Returning user → Face ID → straight to chat
9. ✅ Logo is bigger (340pt)
10. ✅ Sign out → full flush → back to signup

ALL 10 must pass. No archive until all 10 green.
