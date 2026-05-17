# Build 54 P6 Brief — Known Bug Fixes
**Issued by:** Jon | ThunderBase  
**Date:** 2026-05-17  
**Branch:** build34/apns-relay  
**Status:** READY FOR CLI JON

---

## Context
P1, P2, P3, P4, P5, P8 are all committed and pushed on build34/apns-relay (tip: d5559aa).
P6 is known bug fixes from prior builds. Do not touch APNs stack, DeliveryCore, or anything P1-P8 touched.

---

## P6a — SecureField Token Entry UX

**Problem:** Any token entry fields in the app (onboarding, settings, connection setup) use SecureField which hides the text. Typing a 48-character token into a SecureField is painful — you can't verify what you typed.

**Fix:** For any token/password-style field where the user is entering a token (not a real password), add a show/hide toggle button (eye icon) that switches between SecureField and TextField. Standard iOS pattern.

**Files likely affected:** OnboardingView.swift, SettingsView.swift, SignInView.swift, SignUpView.swift

**DO NOT touch:** OnboardingView.swift sign-up flow for the main password field — that's a real password, keep it hidden. Only add show/hide to token entry fields (fields labeled "Token", "Agent Token", "Connection Token", etc.)

---

## P6b — Watchdog Timeout Too Short

**Problem:** Prior builds had a 12-second response timeout for ThunderBase. ThunderBase cold response can take longer, especially under load. Users see timeout errors on first message.

**Fix:** Find the timeout value(s) in the codebase and increase:
- Initial connection timeout: 30 seconds
- Heavy inference response timeout: 60 seconds

Search for: `12`, `timeout`, `timeoutInterval`, `timeoutForRequest`, `timeoutForResource` in the iOS tree.

**Files likely affected:** ThunderCommWebSocketClient.swift, AccountStore.swift, or wherever URLSession/URLRequest timeouts are configured.

---

## Constraints
- DO NOT touch: APNsManager.swift, AppDelegate.swift, DeliveryCore.swift
- DO NOT modify the main password field in OnboardingView — only token fields
- Write files directly, no terminal print
- Do not commit until directed
- Report every file changed and what changed

---

## Deliverable
Report: list of files changed, what changed in each, working tree state. Stop and wait for Jon gate review before committing.
