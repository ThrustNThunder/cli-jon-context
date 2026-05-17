# Build 58 Brief — Final UX Polish
**Issued by:** Jon | ThunderBase
**Date:** 2026-05-17 (for morning of 2026-05-18)
**Base branch:** build56/final (the working Build 57 source)
**Priority:** Ship-blocker fixes only

---

## Context
Build 57 shipped with good infrastructure but missing UX wiring. This brief fixes the remaining issues. 
Mack simulator tests ALL screens before archive. No exceptions.

---

## Fix 1 — Remove Session ID field everywhere

**Files: OnboardingView.swift, AddAgentView.swift**

Delete the "Session ID" / "Agent Session ID" text field entirely. 
Agent connects via token + relay URL only. Session ID is not needed and confuses users.

In OnboardingView Add Agent step: remove the Step 2 section entirely ("Connect using their session ID").
In AddAgentView: remove the Session ID TextField and its section.

---

## Fix 2 — Human appears in their own roster after signup

**Files: UserAccount.swift or SignUpView.swift**

After successful signup, add the user themselves to the relay roster as a human participant.
The relay already tracks connected peers. After signup, the app should connect to the relay 
with the tc-h- token so the user appears as online in their own roster.

In ThunderCommStore or equivalent — after successful auth/signup, call connect() so the WS 
connects to the relay using the tc-h- token. This makes the user visible to agents who connect.

---

## Fix 3 — Agent appears in roster when token is entered

**Files: OnboardingView.swift, AddAgentView.swift**

When user enters an agent's tc-a- token and hits Connect/Save, add that agent to the local 
roster as "pending" or "offline" immediately. Don't wait for them to connect — show them 
in the roster so the user knows the connection was saved.

Store the agent entry in ThunderCommStore with the token. When the agent connects to the 
relay with that token, they'll show as online.

---

## Fix 4 — Face ID at wrong time

**File: ThunderCommApp.swift or AuthGate**

Face ID / biometric auth is firing before the user has created an account. 
It should only fire if the user has a SAVED account (returning user). 
Fresh install = signup screen first, no biometrics.

Check: AuthGate should show SignUpView when no account exists, 
and only trigger biometrics for returning users with saved credentials.

---

## Fix 5 — Logo size on splash screen

**File: SplashView.swift**

Bump logo display size from 260pt to 340pt. Still too small on device.

---

## Fix 6 — Add Agent button in chat goes to wrong screen

**File: ContentView.swift**

The Add Agent button in the main chat UI opens AddAgentView (the old one).
It should open the same Add Agent experience as the onboarding flow — 
or at minimum the same simplified form (token + relay URL, no session ID).

---

## Constraints
- DO NOT touch: DeliveryCore.swift, APNsManager.swift, AppDelegate.swift
- Do not commit until directed
- Report every file changed

## Gate (Mack tests ALL in simulator before archive)
1. ✅ No session ID field anywhere
2. ✅ User appears in roster after signup
3. ✅ Agent appears in roster after token is entered
4. ✅ No Face ID on fresh install
5. ✅ Logo bigger on splash
6. ✅ Add Agent button in chat = same form as onboarding
7. ✅ Chat still loads blank (no hardcoded agents)
8. ✅ Background messages still work (APNs test)
