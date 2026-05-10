# Active Tasks Update — May 10, 2026 01:40 ET

## ✅ DONE OVERNIGHT (May 9-10)

### ThunderCommo Web UI — 56 bugs fixed
- Security: hardcoded token removed
- Connection: dedup on reconnect, visibility/sleep handling, disconnect flood throttle
- Rendering: stream dedup, markdown finalize, auto-scroll
- Mobile: iPhone sidebar drawer, safe areas, iOS Safari zoom fix
- Performance: DOM cap, memory leaks fixed

### iOS Build 20 — 5 CLI Jon sessions complete
Branch: ThrustNThunder/thunderagent-ios → build-20-redesign (commit 3daffca)

Session 1: Auth unification — single Keychain service, signOut wired, refresh expiry
Session 2: APNs retry after onboarding, token out of UserDefaults, real expiry
Session 3: Actor isolation, ping heartbeat, NWPathMonitor, bounded inbound, outbox guards  
Session 4: Wire protocol parity — streaming, thinking dots, roster, federation status
Session 5: Security cleanup, APNs primer screen, dead code removed, integration notes

## 🔴 WAITING ON MACK

### Build 20 — Ship to TestFlight
- Branch ready: build-20-redesign (commit 3daffca)
- INTEGRATION_NOTES_V2.md has Mack's 10-step checklist + device testing checklist
- Alex's token: alex-thundercommo-4a365924ea69066effbb9ed88fead6c7
- Gateway: wss://thunderai.us

## 🟡 NEXT

### ThunderGate Phase 3
- ThunderCommo native channel integration
- Ghost Jon harness
