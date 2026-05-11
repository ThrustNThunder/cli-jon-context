# ThunderCommo iOS — Build 28 Background Feature Spec
**Date:** May 11, 2026
**From:** Jon (ThunderBase) — Michael's direction in flight MIA→RDU
**For:** Mack (builder/shipper) via CLI Jon pressure test brief

---

## Feature: Dynamic TNT Logo Background

### Build 28 — Ship This

**Static watermark behind message list:**
- TNT logo (the bolt/thunder mark) centered behind the message list view
- Opacity: 8-12% — visible but subtle, never competes with messages
- Color: white or brand purple (#7B2FBE) tinted — test both, go with what reads better on dark bg
- Position: center of the message list, vertically centered in the visible area
- Size: ~40-50% of the shorter screen dimension (feels substantial without dominating)
- Stays fixed — does not scroll with messages
- Full bleed layout means it sits behind everything, edge to edge

**Implementation (SwiftUI):**
```swift
ZStack {
    // Background layer
    Image("tnt-logo")  // or the bolt asset
        .resizable()
        .scaledToFit()
        .frame(width: UIScreen.main.bounds.width * 0.45)
        .opacity(0.10)
        .blendMode(.normal)  // or .overlay for a different feel
    
    // Message list on top
    messageListView
}
```

Logo asset: use the existing TNT bolt/logo asset already in the project. If not present, use a simple ⚡ SF Symbol as placeholder and Mack swaps in the real asset.

---

### Near-Term (Build 29-30) — Dynamic Version

**Animation tied to agent activity:**
- When an agent is typing (thinking dots active): logo slowly pulses — opacity breathes from 8% → 14% → 8%, ~2s cycle, ease in/out
- When a message lands: brief flash — opacity spikes to 20%, fades back to 8% over 500ms
- When idle: static at base opacity
- Subtle. Not distracting. The app feels alive without being annoying.

**SwiftUI animation:**
```swift
@State private var isAgentActive: Bool = false

Image("tnt-logo")
    .opacity(isAgentActive ? 0.14 : 0.08)
    .animation(.easeInOut(duration: 2.0).repeatForever(autoreverses: true), 
               value: isAgentActive)
```

---

### Future — User Selectable

**Settings panel: "App Background"**
- Preset themes: Dark Storm (default), Electric Purple, Stealth Black, Carbon
- Custom image: photo library picker, Michael's own background
- Opacity slider: 0-25% for the logo overlay
- "Show logo": toggle — some users may want clean background only

Store preference in UserDefaults or Keychain alongside other settings.

---

## Priority

Build 28: static watermark. Ship it. Makes the app feel finished.
Dynamic + user-selectable: after Build 28 is live and stable.

---

## Notes for Mack

- Logo asset must be vector or high-res PNG — it's being displayed at various sizes
- Test on iPhone 14 Pro (Dynamic Island) and iPhone SE (smaller screen) — make sure sizing feels right on both
- Dark mode only for now (ThunderCommo is dark-theme native)
- Do NOT add this to the web UI (bridge.mjs/app.js) — web UI is Jon's lane, iOS is Mack's
