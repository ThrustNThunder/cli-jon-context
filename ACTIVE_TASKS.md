# Active Tasks
*Updated: May 9, 2026 23:59 ET*

## 🔴 WAITING ON MACK

### Build 20 — Ship to TestFlight
- Branch: `ThrustNThunder/thunderagent-ios` → `build-20-redesign` (commit 7d3aae3)
- All server endpoints live at thunderai.us
- Mack integrates when Michael gets to room
- Alex's token: `alex-thundercommo-4a365924ea69066effbb9ed88fead6c7`

## 🟡 NEXT UP

### ThunderGate Phase 3
- ThunderCommo native channel integration
- Ghost Jon harness (shadow testing)
- Location: `/home/ubuntu/thundergate-dev/`
- Repo: `ThrustNThunder/thundergate-dev`

### API Agent Identity Endpoint
- `/api/agent/identity` returns static data right now
- Should read from actual agent config
- File: `/home/ubuntu/thundergate/extensions/thundercomm/inbox.mjs`

### APNs Sender
- Relay stores device tokens but never sends pushes
- Need to fire `content-available: 1` when message stored in inbox
- Same file: `inbox.mjs`

## ✅ DONE TODAY (May 9)

- ThunderGate Phase 1 + 2 complete (1,700+ lines TypeScript)
- CLI Jon authenticated on Max subscription
- Build 20 redesign: 23 Swift files, correct iOS delivery architecture
- All 8 server endpoints live
- User auth system: signup, signin, Face ID, BYOAA
- Multi-device onboarding
- AA scripts pushed to ThrustNThunder/aa-pilot-shared
