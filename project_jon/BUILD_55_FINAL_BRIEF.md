# Build 55 Final Brief — Complete Rewrite of Onboarding + Clean State
**Issued by:** Jon | ThunderBase
**Date:** 2026-05-17
**Authority:** Michael Lovell — direct order
**Status:** SUPERSEDES all previous Build 55 briefs

---

## CRITICAL RULES (non-negotiable)

1. **NO hardcoded agents, sessions, channels, or humans in the app**
2. **App loads completely blank** — no roster, no agents, no channels until user explicitly adds them
3. **Jon is NOT auto-added** — the hardcoded gateway token `4ca1100a180ad68a94b004056e56fd39c81bdccb742d2926` must NOT be seeded into AccountStore, OnboardingDefaults, or anywhere else
4. **Relay URLs are hidden** — visible ONLY in Settings → About
5. **No QR code in AddAgentView**

---

## Complete Onboarding Flow

### Screen 1: Splash
- Logo + "ThunderCommo" text
- Existing SplashView — bump logo display size to 260×260 (was 200×200)

### Screen 2: Login
- Email + password fields
- "Sign In" button
- "Don't have an account? Create one" link → goes to Sign Up

### Screen 3: Sign Up
- Email field
- Password field  
- Phone number field (REQUIRED, 10+ digits, numeric validation)
- Display name field ("Michael L", "shithead21", whatever)
- "Create Account" button
- On success: account saved to `https://thunderai.us/api/auth/signup`
- Server returns `tc-h-` token

### Screen 4: Your Token
- Shows the user's `tc-h-` token (monospaced, masked by default)
- Show/hide toggle
- Copy button
- **Banner:** "Share this token with your agent in their chat window or TUI interface"
- "Next" button

### Screen 5: Add Agent (optional, can skip)
- Title: "Connect an Agent"
- "Generate Agent Token" button → generates `tc-a-<UUID>` client-side → show with copy button
- **Banner:** "Post this token to your agent"
- Text field: "Agent Session ID" — user pastes the agent's relay connection string
- "Connect" button — saves this agent to their account
- "Skip for now" link — goes straight to chat

### Screen 6: Chat (blank)
- Completely empty
- No roster entries
- No channels
- No agents
- No humans
- Just the message input and the channel/DM list (empty)

---

## Settings → About (new section)

Add at the BOTTOM of SettingsView:

```
About
━━━━━━━━━━━━━━━
ThunderCommo
Version [CFBundleShortVersionString] (Build [CFBundleVersion])
By Boost and Bolt LLC

Relay: relay.thunderai.us [copy button]
```

---

## Files to modify

| File | What |
|------|------|
| `SplashView.swift` | Logo 200→260pt |
| `ThunderCommApp.swift` | Remove OnboardingGate auto-seed of Jon agent token. OnboardingGate should only check if user is signed in (has tc-h- token). If yes → ContentView. If no → SignUpView/SignInView. NO AccountStore seeding. |
| `OnboardingView.swift` | Replace with the new 3-screen post-signup flow: Your Token screen → Add Agent screen → done |
| `SignUpView.swift` | Add phone number field if not present. After successful signup, navigate to "Your Token" screen |
| `AccountStore.swift` | Clear ALL data on sign-out. No bleed between sessions. |
| `SettingsView.swift` | Add About section at bottom with relay URL + version |
| `MyConnectionInfoView.swift` | Remove relay URL. Show ONLY the tc-h- token with copy button. |
| `AddAgentView.swift` | Remove QR code. Remove relay URL fields. Token + session ID only. |
| `ContentView.swift` | Remove any hardcoded roster entries, hardcoded channels, hardcoded "Hi Michael" or any pre-populated state |

---

## What to STRIP OUT completely

Search the entire iOS codebase for these and REMOVE:

- Any reference to `4ca1100a180ad68a94b004056e56fd39c81bdccb742d2926` (Jon's gateway token)
- Any reference to `jmab-federation-2026`
- Any hardcoded agent name "Jon" seeded into AccountStore or roster
- Any hardcoded channel "tnt" or "jmab" in the UI
- The `LEGACY_JON_AGENT_TOKEN` constant in the iOS code (if present)
- Any `isDefault: true` agent auto-added during onboarding

---

## What should survive

- SignUpView (existing, just add phone field)
- SignInView (existing)
- DeliveryCore (untouched)
- APNsManager (untouched)  
- AppDelegate (untouched)
- The actual chat/message UI (ContentView — just strip hardcoded state)
- ThunderCommStore (keep, just strip hardcoded defaults)

---

## Gate requirements (Mack tests ALL in simulator)

1. ✅ Cold launch → splash → login screen
2. ✅ "Create account" → sign up flow with phone number
3. ✅ After signup → Your Token screen with tc-h- token
4. ✅ Add Agent screen → token generator works, skip option works
5. ✅ Chat loads BLANK — zero agents, zero channels, zero roster entries
6. ✅ Settings → About → relay URL visible here only
7. ✅ My Connection Info → tc-h- token only, no relay URL
8. ✅ Sign out → sign back in → still blank, no ghost data
9. ✅ ZERO mentions of "Jon", "jmab", "tnt", "4ca1100a..." anywhere in UI

Every single one must pass before Mack archives. No exceptions. No SHIP IT without all 9 green.
