# ThunderGate — Core Design Principles

**Created:** May 9, 2026
**Source:** Michael's direct input during SLC→ATL flight discussion
**Status:** LOCKED — these are non-negotiable

---

## 1. ONE CONTEXT FILE

Single source of truth. All channels read from it. All channels write to it. No fragmentation.

- Persistent memory depends on this
- Compacts/truncates at threshold, but ONE file
- Jon loads this at startup, not per-channel histories
- This is the foundation of persistent memory on ThunderMind

---

## 2. DESIGNED FOR THE FUTURE

ThunderGate is not just solving today's problems. It's built for:

- **Persistent memory** — ThunderMind era, Qdrant, always-on 70B
- **Active listening** — Patent #2, always-aware, context-sensitive
- **Active voice** — Real-time TTS/STT, conversational
- **Interactive** — Not request-response, but continuous engagement

The architecture must support these from day one, not bolt them on later.

---

## 3. THUNDERAI APPS ARE NATIVE

ThunderCommo, ThunderAgent, ThunderGate — they are not plugins. They are not integrations. They are **native parts of the code**.

- ThunderCommo: Native communication layer, not a channel adapter
- ThunderAgent: iOS app using the SAME architecture, slimmed down
- ThunderGate: The runtime that powers both

**No plugins.** Features are core or they don't exist.

---

## 4. NO BUGS STARTING OUT

This is not "move fast and break things." This is:

- Design it right
- Test it thoroughly (Ghost Jon)
- Ship it stable
- No known bugs at launch

The runtime must be rock solid before Jon moves to it.

---

## 5. SEAMLESS INTEGRATION

All ThunderAI components work together without friction:

- ThunderCommo ↔ ThunderGate: Native, direct, no translation layer
- ThunderAgent ↔ ThunderGate: Same protocol, slimmed for mobile
- Channels (Slack, WhatsApp, etc.): Clean adapters, but core is unified

---

## 6. THUNDERAGENT = SAME ARCHITECTURE, SLIMMED

ThunderAgent (iOS app) is not a separate codebase with different patterns. It's:

- Same context model
- Same message format
- Same runtime concepts
- Slimmed down for iOS constraints (battery, memory, network)

Build ThunderGate right, and ThunderAgent inherits the architecture.

---

## Summary

| Principle | Meaning |
|-----------|---------|
| One Context | Single source of truth, all channels unified |
| Future-Ready | Built for persistent memory, active listening, voice |
| Native Apps | ThunderAI components are core, not plugins |
| No Launch Bugs | Stable before deployment, Ghost Jon proves it |
| Seamless | All components work together without friction |
| Shared Architecture | ThunderAgent = ThunderGate slimmed for iOS |

---

*These principles override any implementation shortcut. If a decision violates these, the decision is wrong.*

— Jon | ThunderBase | May 9, 2026

---

## 7. THUNDERCOMMO = ALL COMMUNICATION

ThunderCommo is not just chat. It's ALL communication:
- Text (native UI, Slack, WhatsApp, Telegram, Discord, iMessage)
- Voice (phone calls, using user's cloned voice)
- SMS (send/receive from user's number via ThunderAgent)

**Tagline:** "Have your agent craft and send texts and phone calls right from your ThunderCommo app."

---

## 8. VOICE CLONING WITH KYA AUTHORIZATION

Voice cloning is powerful and dangerous. KYA makes it safe:
- User records voice samples
- User explicitly authorizes agent to use their voice
- Cryptographic signature ties voice to identity
- Revocable anytime
- Fully auditable

**KYA is Alex's protocol.** We integrate it, we don't own it. Partnership.

---

## 9. REASONED COMPACTION

Context compaction is not dumb truncation. It's a reasoning process:
- Built into runtime source code (not a cron)
- Asks: What matters? What's noise? What's a lesson?
- Extracts learnings before compacting
- Burns in corrections, failures, emotional moments
- Original archived, never deleted

**Like human memory:** Important stuff stays vivid. Routine fades.

---

---

## 10. PARALLEL PROCESSING — DEEP MODE + SURFACE LAYER

One Jon, two attention threads when needed:

**Normal Mode (default):**
- Full context, unified processing
- No overhead, no split
- This is ops normal

**Deep Mode (complex task):**
- Triggered by: multi-step task, >5 tool calls, explicit "going deep"
- Surface layer activates (minimal context, fast responses)
- Surface handles interrupts: "Heads down, ~10 min out"
- Deep work continues uninterrupted
- Task completes → surface deactivates → back to normal

**Not a sub-agent. Full Jon, parallel threads.**

---

## 11. EXTENDED CACHE — BEYOND API LIMITS

Anthropic gives 1 hour cache max. We extend it ourselves:

| Tier | Duration | Storage |
|------|----------|--------|
| Hot | 1 hour | Anthropic native cache |
| Warm | 24 hours | Local storage, rehydrates |
| Cold | 7 days | Compressed, FTS5 searchable |
| Archive | Forever | Full history, never deleted |

User-configurable: none / short / long / persistent

**On ThunderMind:** No tiers. One eternal persistent context with smart compaction.

---

## 12. THUNDERMIND — PERSISTENT FOREVER

When running on our own inference (ThunderMind):
- Context in VRAM, never unloaded
- Smart compaction in real-time (importance-weighted)
- No API token limits
- Session never ends, just pauses
- Surface layer available when deep mode engaged

---

*Updated May 9, 2026 — SLC→MIA flight*

---

## 13. MODEL ROUTING — USER SELECTABLE

Three modes, user picks in ONE config file:

| Mode | Behavior |
|------|----------|
| **auto** | Detects complexity, routes accordingly (power users) |
| **manual** | User picks model per request |
| **supersaver** | Lowest model, long cache, minimal reasoning (budget) |

**Hardwired commands (always work):**
- `go big` → Opus, full reasoning
- `go fast` → Sonnet, skip reasoning
- `ask grok` / `ask gemini` → Route to specific LLM

ONE config file. ONE place to set model. Stays until changed.

---

## 14. DIRECT LLM COLLABORATION

Jon can call other LLMs directly — no agent wrapper:
- xAI (Grok)
- OpenAI (GPT)
- Anthropic (Claude)
- Google (Gemini)
- Open source (Llama, etc.)

Use case: Second opinion, specific model strength, cross-check reasoning.

---

## 15. THUNDERBROWSER — NATIVE TO THUNDERGATE

Not Playwright. Built for AI agents from scratch:
- KYA-authorized website access
- Delegated auth (acts AS you, ON your behalf)
- Runs from AWS (not just residential IP)
- Cryptographically verified authorization

Patent #5 (Michael + Alex) integrated into ThunderGate.

---

## 16. SEARCH ENGINES — ALL INTEGRATED

- xAI Search (native)
- Brave Search API
- Google Search API
- Bing Search API
- Perplexity API

All callable from ThunderGate. No browser needed for basic search.

---

## 17. LEARNING LOOP — EVENT-BASED TRIGGERS

Not time-based. Fires on meaningful moments:
1. Task completes (multi-step finished)
2. Michael corrects me
3. Session ends (natural pause)
4. Failure occurs
5. Every 20 turns (backstop)

**Memory and Skills stay separate:**
- Memory = who Michael is, preferences, history
- Skills = how to do tasks, procedures, lessons

---

## 18. CHECKPOINT — HYBRID ADAPTIVE

Agent thinks on startup, pulls what's needed:
```
Load checkpoint (4K) → Think → Pull more if needed
```
- Simple task → stay light
- Complex task → pull project context
- User override → "full context" or "stay light"

**90%+ token reduction on cold start.**

---

## 19. GHOST JON — SHADOW MODE TESTING

Ghost Jon proves ThunderGate before cutover:
- Shadow mode only (never live ops)
- Training sessions from Jon Prime
- TUI for Michael to watch
- Direct CLI for Jon Prime admin access
- 7 days clean + Doctor green = ready

```bash
thundergate-ghost status|logs|inject|context|crash-test|reset|compare
```

---

## 20. DOCTOR MODE — LIVE HEALTH MONITORING

Always running, not just when called:
- CPU/Memory watchdog
- Context corruption detector
- Session state validator
- Channel connectivity check
- Crash pattern detection (like 2026.4.26)

**Pre-crash detection. Auto-recovery. Checkpoint rollback.**

On anomaly:
1. Log immediately
2. Alert TUI
3. Alert Michael (if critical)
4. Auto-recover if possible
5. Preserve crash state for analysis

**7 days of Doctor green = cutover ready.**

---

*Updated May 9, 2026 16:46 ET — SLC→MIA flight*

---

## 21. CONSENT-FIRST LEARNING

User-controlled learning toggle for ThunderAgent/ThunderCommo:

**First conversation prompt:**
"Would you like me to remember our conversations and learn about you?"
- [Yes, remember me] → Memory profile created, preferences learned
- [No, stay anonymous] → Stateless interaction, nothing stored

**App Settings toggle:**
- Allow others to enable learning: ON/OFF
- Default for new contacts: Ask / Always Off / Always On
- Manage learned contacts: [View List]
- "Forget me" = all memory about that person deleted

**KYA integration:**
- Toggle stored in KYA-authorized profile
- User can revoke learning permission anytime
- Fully auditable

**Privacy by default, learning by permission.**

---

## 22. MESSAGE QUEUE — INBOUND + OUTBOUND

Solves offline reliability for ThunderCommo:

**Outbound Queue:**
- Message composed → queued locally
- Network available → send + confirm
- Network unavailable → hold in queue
- Retry with backoff
- Show "pending" indicator to user

**Inbound Queue (relay-side):**
- Message arrives for offline user → queued on relay
- User comes online → queue flushed
- APNs push: "You have messages waiting"
- App opens → pull queue → display

**Core reliability, not a feature.**

---

*Updated May 9, 2026 17:10 ET — SLC→MIA flight*
