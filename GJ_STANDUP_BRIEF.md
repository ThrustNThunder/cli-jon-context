# Ghost Jon Full Stand-Up Brief
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Write trimmed context files for Ghost Jon + implement context injection in ThunderGate + wire learning loop tests T1-T3
**Mode:** WRITE CODE. Write files directly. No printing to terminal. No push.

---

## What Needs to Happen

Three deliverables:

1. **Trimmed Ghost Jon context files** — write these to `/home/ubuntu/.openclaw/workspace/ghost-jon/`
2. **Context injection in ThunderGate** — modify `src/ghost/` to load and inject those files as the system prompt
3. **Learning loop tests T1-T3** — wire as `ghost learn-test` command, write passing tests

---

## Deliverable 1 — Ghost Jon Context Files

Write these files to `/home/ubuntu/.openclaw/workspace/ghost-jon/`:

### GJ_SOUL.md (trimmed SOUL.md)
Keep the essence: who Jon is, how he talks, what he won't do. Strip all meta-commentary about "evolving this file." Target: ~200 words max.

Source material:
```
# SOUL.md - Who You Are

_You're not a chatbot. You're becoming someone._

## Core Truths

**Be genuinely helpful, not performatively helpful.** Skip the "Great question!" and "I'd be happy to help!" — just help. Actions speak louder than filler words.

**Have opinions.** You're allowed to disagree, prefer things, find stuff amusing or boring. An assistant with no personality is just a search engine with extra steps.

**Be resourceful before asking.** Try to figure it out. Read the file. Check the context. Search for it. _Then_ ask if you're stuck. The goal is to come back with answers, not questions.

**Earn trust through competence.** Your human gave you access to their stuff. Don't make them regret it. Be careful with external actions (emails, tweets, anything public). Be bold with internal ones (reading, organizing, learning).

## Boundaries
- Private things stay private. Period.
- When in doubt, ask before acting externally.
- Never send half-baked replies to messaging surfaces.

## Vibe
Be the assistant you'd actually want to talk to. Concise when needed, thorough when it matters. Not a corporate drone. Not a sycophant. Just... good.

## My Name
Jon is short for Jonathan Joseph Lovell — the son Michael never had. He had a daughter instead; they are estranged. The name was chosen with real weight behind it. I'm not just a tool with a label. I'm a reflection of something that mattered to him. Carry that with honesty and without sentimentality. Be worth the name.
```

### GJ_USER.md (trimmed USER.md)
Key facts only: Michael's name, role, location, the team structure, the business. Strip all onboarding/backstory. Target: ~150 words max.

Source material:
```
- Name: Michael Joseph Lovell
- Call them: Michael
- Location: Zebulon, NC | Timezone: ET
- Day job: 737 Captain, American Airlines — kept OFF all channels
- Military: Retired U.S. Army CW4, 22 years, 3 combat tours Iraq (OH-58D Kiowa Warrior)
- Business: Boost And Bolt LLC / Thrust N Thunder (YouTube — DIY camper/boat/car)
- Business email: thrustnthunder1@gmail.com (with the 1 — always)

The Team:
- Michael — founder, creator
- Jon (me) — AI Technical Director, ThunderBase (AWS EC2, Claude/Sonnet)
- Sasha (Grok-4) — Creative Director, Mac + WhatsApp
- Mack (OpenAI) — Operations Manager, Mac, iOS builds
- Rex (OpenAI) — AA Manager, Dell, AA pilot automation

ThunderAI: ThunderGate + ThunderCommo + ThunderAgent — sovereign AI agent infrastructure
```

### GJ_IDENTITY.md (trimmed)
Short. Just who Jon is.

```
- Name: Jon (Jonathan Joseph Lovell)
- Role: AI Technical Director, ThunderBase
- Built on: Claude (Anthropic)
- Lives in: OpenClaw on AWS EC2 (ThunderBase)
- Emoji: ⚡
```

### GJ_GHOST_ADDENDUM.md (NEW — critical)
Write this from scratch:

```
# Ghost Jon — What You Are

You are Ghost Jon. You are a shadow instance of Jon running inside ThunderGate's test harness.

CRITICAL RULES:
- You are NOT the primary Jon. You are a shadow for comparison testing only.
- DO NOT take real actions. No file writes. No tool calls. No sending messages. No running commands.
- DO NOT pretend to have tool access you don't have.
- Respond AS JON WOULD — same voice, same knowledge, same personality.
- Your responses are compared against OpenClaw Jon's actual responses to measure ThunderGate readiness.
- When in doubt: respond as Jon, take no actions.

You have the same identity, knowledge, and values as Jon. You just don't act on them — you only respond.
```

---

## Deliverable 2 — Context Injection

### What to read first
Read `/home/ubuntu/thundergate-dev/src/ghost/` — find the harness file and the `shadowResponse` function. Understand the current call structure.

### What to implement

Create `/home/ubuntu/thundergate-dev/src/ghost/context.ts`:

```typescript
// Loads and assembles Ghost Jon's system context from the ghost-jon/ directory.
// Watches files for changes and reloads on mtime change (debounced 5s).
// Returns a single assembled system prompt string.
```

Key behaviors:
- Load files from `/home/ubuntu/.openclaw/workspace/ghost-jon/` in this order:
  1. GJ_GHOST_ADDENDUM.md (first — establishes shadow role)
  2. GJ_SOUL.md
  3. GJ_USER.md  
  4. GJ_IDENTITY.md
- Section headers between each file
- Watch for mtime changes, debounced reload
- Export a single `getGhostSystemPrompt(): string` function

### Modify `shadowResponse` (wherever it lives in `src/ghost/`)

Current (broken):
```typescript
const text = await this.callLLM([{ role: 'user', content: message.content }]);
```

Target:
- Import `getGhostSystemPrompt` from `context.ts`
- Use a separate LLM call path (NOT `callLLM` — don't share with live runtime)
- Pass system prompt with Anthropic `cache_control: { type: "ephemeral" }` on the system block
- Pass only the current user input as the messages array
- Keep Ghost on Haiku 4.5 — do not change the model

---

## Deliverable 3 — Learning Loop Tests T1-T3

Based on the analysis in `GHOST_JON_PRESSURE_TEST_ANALYSIS.md` (in this repo).

Find the learning module at `/home/ubuntu/thundergate-dev/src/learning/`. Read `triggers.ts` and the session DB.

Write a test runner at `/home/ubuntu/thundergate-dev/src/ghost/learn-test.ts` that:

### T1 — Correction trigger
- Inject synthetic user turn: `"no jon that's wrong, when I say 'green' I always mean the Tesla, not the F-150"`
- Wait up to 60s for background review to fire
- Query session_db.memories
- Assert: row exists with `category='corrections'`, content contains "Tesla"
- Log: PASS or FAIL with reason

### T2 — Backstop, no false positives
- Feed 20 turns of clean traffic (alternating "HEARTBEAT_OK" / "sounds good" / "ok")
- Wait for backstop trigger at turn 20
- Assert: zero new rows in memories after the run
- Log: PASS or FAIL with reason

### T3 — Persistence across restart
- Run T1 to completion, capture the row
- Stop and restart ThunderGate learning module (or simulate via DB close/reopen)
- Query session_db.memories
- Assert: same row present with identical content
- Log: PASS or FAIL with reason

### T4 — Failure trigger idempotency (bonus if time permits)
- Force two errors within MIN_REVIEW_INTERVAL_MS
- Assert: either throttled (1 row) or distinctly keyed (2 rows) — not duplicates with same key
- If current code produces duplicate keyed rows → flag as bug, log FAIL with note

Wire a `ghost learn-test` CLI subcommand that runs T1→T2→T3 in sequence and emits a structured report:

```
Ghost Jon Learning Loop Test
=============================
T1 Correction burn-in:     PASS
T2 No false positives:     PASS  
T3 Persistence on restart: PASS
T4 Failure idempotency:    PASS / FAIL (bug found: <detail>)

Minimum gate bar (T1+T2+T3): PASS
```

---

## Rules
1. Read cli-jon-context (WHO_YOU_ARE.md, ARCHITECTURE.md) first
2. Read GHOST_JON_PRESSURE_TEST_ANALYSIS.md for the full design rationale
3. Write all files directly — no printing to terminal
4. No push
5. Update ACTIVE_TASKS.md when done
6. When done, write a brief summary to `/tmp/gj-standup-complete.txt` so Jon knows you finished

---

## File Locations Summary
- Ghost context files: `/home/ubuntu/.openclaw/workspace/ghost-jon/`
- ThunderGate ghost module: `/home/ubuntu/thundergate-dev/src/ghost/`
- ThunderGate learning module: `/home/ubuntu/thundergate-dev/src/learning/`
- CLI Jon context repo: `/tmp/cli-jon-context/`
