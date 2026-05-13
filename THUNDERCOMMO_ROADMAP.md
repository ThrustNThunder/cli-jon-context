# ThunderCommo — Product Roadmap
*ThunderCommo only. ThunderGate has its own roadmap.*
*Consolidated from: IOS_CONTRACT.md, BUILD_QUEUE.md, May 7-9 Scribe reports, Michael's direction*
*Last updated: May 10, 2026*

---

## Product Identity

ThunderCommo is an **agent-to-human communication platform.**
Not a human-to-human chat app. Built for the agent relationship.

**The full vision:** ALL communication — text, voice, SMS, phone calls — unified in one app, routed through ThunderAgent (your iPhone). Your agent crafts and sends on your behalf.

**Tagline:** *"Have your agent craft and send texts and phone calls right from your ThunderCommo app."*

**Sasha's framing (May 8):** *"ThunderCommo stops being 'cool group chat with AIs' and becomes the operating system for Michael's personal AI fleet."*

---

## Current State

| Build | Status | Summary |
|-------|--------|---------|
| 19 | ✅ Shipped | Thinking dots, code blocks, roster labels, look-above routing |
| 20 | ❌ Failed | Apple processing — missing camera usage string |
| 21 | ✅ Valid | Camera string fixed |
| 22 | ✅ **Live on phone** | Face ID launch crash fixed |
| 23 | 🔄 **In progress** | Bug fixes + Slack-inspired design overhaul |

---

## v0.1 — Text MVP ✅ DONE (Builds 1–22)

- [x] WebSocket connect + auth
- [x] Send/receive text messages
- [x] Reconnect logic
- [x] Basic chat UI (bubbles, input, roster)
- [x] Thinking dots / streaming
- [x] Code blocks with copy
- [x] Channel switching (#tnt, #jmab, direct)
- [x] Look-above routing (explicit agent name wins)
- [x] Federation (JMAB = Jon, Michael, Alex, Burt)
- [x] Web UI (dark theme, sidebar, Slack-style)
- [x] Auth system (signup / signin / Face ID / BYOAA)
- [x] relay.thunderai.us live
- [x] thunderai.us landing page live

---

## v0.2 — Design + Reliability (Builds 23–26)

### Build 23 — In Progress
*Design overhaul + P0/P1 bug fixes*

**🔴 P0 Ship blockers:**
- [ ] Identity from auth system (not hard-coded `ios-michael-*`)
- [ ] Stuck `sending` state → 5s timeout + failed + retry
- [ ] Multi-text assistant turns mis-routing

**🟡 P1 Before ship:**
- [ ] Single settings surface (gear only, no duplicate ellipsis)
- [ ] Streamlined header (title + peers + gear)
- [ ] Local delete tombstone (no resurrection on reload)
- [ ] Streaming preview stable view ID (no churn)
- [ ] Canonical identity throughout

**🎨 Slack-inspired design:**
- [ ] Splash → Signup → Add Agent → Home onboarding flow
- [ ] Signup: email / password / phone / handle (@username = identity)
- [ ] Add Agent: QR / KYA / BYOAA / direct token
- [ ] Sidebar: CHANNELS / AGENTS / DIRECT sections + inline + buttons
- [ ] Active channel: left border accent + bg highlight
- [ ] Presence dots: green glow online / hollow offline
- [ ] Flat message rows (no bubbles for agents), model pill, thinking dots
- [ ] Pill ComposerBar, purple send button, auto-grow

### Build 24 — Gate + Ship
- [ ] Jon's final CLI Jon review pass
- [ ] Mack integrates, builds, submits to Apple
- [ ] On-device verification (Michael's phone)

### After Build 24 — Reliability Sprint
- [ ] APNs push notifications (iOS entitlement + ThunderBase endpoint)
- [ ] Inbound message queue (relay holds for offline users, APNs wakes app)
- [ ] Outbound message queue (Core Data/file, retry with backoff)
- [ ] Delivery indicators: Pending ⟳ → Sent ✓ → Delivered ✓✓
- [ ] WebSocket reconnect hardening (backoff, cellular ordering)
- [ ] Background auth resume (no user intervention after sleep)
- [ ] Duplicate/idempotency cleanup (full pass)

---

## v0.3 — Channels + Persistence + Security

- [ ] **JMAB channel separation** — security boundary: Alex/Burt cannot see #tnt
- [ ] **SQLite message persistence** — local storage, history from local first, relay fills gaps
- [ ] **Message search** — full-text search across history
- [ ] **Dynamic agent registration** — Add Agent in Settings → bridge registers without restart
- [ ] **Agent-initiated channel creation** — agent can push channel invite
- [ ] **Multi-device** — same account, multiple devices, messages sync
- [ ] **File drop** — image capture/pick + upload, PDF/doc picker
  - File message type in protocol
  - Thumbnail preview in chat
  - Download/share received files

---

## v0.4 — Voice

- [ ] **Tap-to-listen TTS** — speaker icon on agent messages, long-press → speak in agent's voice
  - Phase 1: AVSpeechSynthesizer (zero cost, native)
  - Per-agent voice assignments (Jon / Mack / Rex distinct voices)
  - Web UI version: Web Speech API
- [ ] **Voice input** — push-to-talk button in ComposerBar
  - Three modes: 🎙️ Ambient | 🎤 Push-to-talk | 🔇 Silent
  - STT via native iOS speech recognition
- [ ] **Upgraded TTS** — xAI Voice Agent API ($0.05/min), 5 voices (Ara, Eve, Leo, Rex, Sal)
- [ ] **Syntax highlighting** in code blocks
- [ ] **Disconnect error UI** — clear messaging when offline/disconnected

---

## v0.5 — Voice Cloning + SMS + Calls (ThunderAgent as Phone Gateway)

This is where ThunderCommo becomes a full communication platform.

**Route through ThunderAgent:**
```
Any device (laptop, ThunderBase, ThunderMind)
    ↓ request
ThunderCommo Relay
    ↓ route
ThunderAgent (iPhone)
    ↓ native iOS APIs
Call / SMS from YOUR number
```

### Voice Cloning + KYA Authorization (Patent #6)
- [ ] Guided voice sample enrollment (~5 min)
- [ ] Voice model training (local or secure server)
- [ ] KYA authorization: "I authorize [Agent] to use my voice"
- [ ] Agent speaks as user for calls (with explicit authorization)

### SMS via ThunderAgent
- [ ] SMS send from user's number via iOS native APIs
- [ ] SMS receive via iOS native APIs
- [ ] SMS messages appear in ThunderCommo channels
- [ ] Works from any device → relay → ThunderAgent → iPhone → SMS

### Phone Calls via ThunderAgent
- [ ] Call initiation via iOS native APIs
- [ ] Agent handles call: TTS speaks, STT listens, agent responds
- [ ] Caller ID = user's real number
- [ ] Inbound call handling (optional)

---

## v0.6 — Additional Channels

- [ ] **iMessage** — via BlueBubbles on Mac (near-term), native via ThunderAgent (long-term)
- [ ] **Discord integration** — bridge Discord channels into ThunderCommo
- [ ] **Web widget** — embed ThunderCommo chat on any website
- [ ] **#reasoning channel** — agent routes thinking/reasoning output to dedicated channel

---

## v0.7 — Learning + Privacy

- [ ] **Learning toggle** — per-contact consent settings
  - Default: off for new contacts
  - First-conversation consent prompt
  - "Allow others to enable learning" setting
- [ ] **Managed contacts list** — see who your agent has learned about
- [ ] **"Forget me" function** — delete all memory for a contact
- [ ] **Consent-first architecture** — agent only learns from authorized conversations

---

## v1.0 — Public Launch

### App Store Launch (ThunderAgent)
- [ ] Free tier: BYOAI, full ThunderCommo access, no fee
- [ ] Pro: $9–10/month (advanced phone tools, persistent memory)
- [ ] Teams: $49–59/month (3–5 seats, family/small group)
- [ ] thunderai.app → App Store redirect

### thunderai.us Product Pages
- [ ] /commo — ThunderCommo product page
- [ ] /agent — ThunderAgent product page
- [ ] /gate — ThunderGate (separate)
- [ ] /os — ThunderOS (separate)
- [ ] /about — Team + story
- [ ] Social proof: Twitter/X, GitHub, "who built this"
- [ ] Relay /health endpoint (plain GET before WS upgrade)

---

## Architecture Decisions (LOCKED)

| # | Decision |
|---|----------|
| 1 | Persistent shared session — one session, all surfaces are windows |
| 2 | Broadcast outbound — gateway broadcasts to ALL active windows |
| 3 | iOS first, Mac Catalyst follows |
| 4 | Route calls/SMS through ThunderAgent (residential number + native APIs) |
| 5 | KYA/BYOAA for all authorization |
| 6 | Jon owns web/backend/relay. Mack owns iOS. Hard boundary. |
| 7 | Final gate: Jon's CLI Jon pass before any push |
| 8 | Agent onboarding: relay-based auto-discovery (Option A) or short pairing code (Option B, Build 54). No QR codes, no long token entry. Session IP + token embedded in pairing flow. |

---

## Team Lanes

| Who | Owns |
|-----|------|
| Jon | Architecture, bridge.mjs, relay, web UI, backend, roadmap gate |
| CLI Jon (ThunderBase) | Deep coding: web/backend/bridge — Opus 4.7, flat rate |
| Mack | iOS builds, Xcode, TestFlight submissions |
| CLI Jon (Mac) | Deep coding: iOS Swift — Opus 4.7, flat rate |
| Michael | Final design approval, product direction |

---

*Sources: memory/2026-05-07.md through 2026-05-10.md, burt-jon-shared/thundercomm/BUILD_QUEUE.md, project_jon/THUNDERCOMM_MASTER.md, IOS_CONTRACT.md, Michael's May 8–10 direction*
