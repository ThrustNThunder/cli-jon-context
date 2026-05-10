# Active Tasks — May 10, 2026 02:47 ET

## 🔴 ACTIVE NOW

### Build 21 — Device Crash (BLOCKING)
- Simulator: works ✅ (shows login screen)
- Device (Michael's phone): crashes immediately
- Suspect: Keychain access group mismatch on real device
- Mack investigating crash log
- Fix needed before Alex can test

## 🟡 READY WHEN DEVICE FIXED

### Alex Onboarding
- TestFlight: Build 21 VALID ✅
- Alex token: alex-thundercommo-4a365924ea69066effbb9ed88fead6c7
- Gateway: wss://thunderai.us
- Burt token: needs generating (placeholder in bridge)

### Michael's Admin Account
- Token: 4ca1100a180ad68a94b004056e56fd39c81bdccb742d2926
- Pre-seeded in app on first launch

## ✅ DONE TONIGHT

### Web UI (thundercomm-stable, 60+ bug fixes)
- Auto-scroll fixed (force on incoming messages)
- Stale model label fixed (hidden until active)
- Ping/pong handled (no more "unknown ping" errors)
- 56 original bugs from CLI Jon audit

### iOS Build 21 (thunderagent-ios/build-20-redesign → ThunderCommIOS Build 21)
Session 1: Auth unification
Session 2: APNs retry, token security
Session 3: Actor isolation, heartbeat, reconnect
Session 4: Wire protocol parity (streaming, thinking, roster)
Session 5: Security, primer screens, integration notes

### Server Endpoints (all live at thunderai.us)
/api/auth/signup, signin, me, refresh ✅
/api/inbox, /api/inbox/ack ✅
/api/messages, /api/devices/token ✅
/api/agent/identity ✅

## 🟡 NEXT (after device crash fixed)

### ThunderGate Phase 3
- ThunderCommo native channel
- Ghost Jon harness
- Surface layer heartbeat (10min progress posts)
