# Ghost Jon Pressure Test Brief
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Diagnose Ghost Jon's 0% match rate + design fix + test the Hermes learning loop
**Mode:** Analysis + recommendations. NO CODE unless specifically asked.

---

## The Problem

Ghost Jon is running (Day 1 of 7-day clean gate) but scoring **0% match rate** on 241 samples.

Looking at the logs, the root cause is obvious:

Ghost Jon has **zero context**. It's responding like a blank Claude instance that just received a random suspicious message. Examples from the log:

- Michael says "Jon?" → Ghost Jon says "I can see this is a chat message from Michael L... is there something specific you'd like me to help with regarding this message?"
- Michael talks about BYOAA → Ghost Jon says "I should note that the sender metadata is marked as untrusted" and asks security clarifying questions
- Michael says "write it all down bud" → Ghost Jon treats it as an unverifiable instruction and refuses

Ghost Jon has no idea who Michael is, what ThunderAgent is, what BYOAA is, what #tnt is, or that it IS Jon. It's treating every message as isolated external input from an unknown source.

OpenClaw Jon (me) loads context on startup: SOUL.md → USER.md → MEMORY.md → AGENTS.md → daily memory files. Ghost Jon loads nothing.

**The fix is context injection** — Ghost Jon needs the same startup context I have.

---

## What You Need to Analyze

### 1. Context Injection Architecture

Ghost Jon is a ThunderGate harness running Haiku 4.5 against the OpenClaw session transcript. It intercepts messages and generates shadow responses for comparison.

The question is: **how should context be injected into Ghost Jon?**

Options to evaluate:
- **Option A: System prompt injection** — inject SOUL.md + USER.md + MEMORY.md + AGENTS.md as the system prompt for every Ghost Jon inference call
- **Option B: First-turn injection** — inject context as a synthetic "first turn" before each message
- **Option C: Context file loading** — ThunderGate loads context files at Ghost Jon startup, keeps them in the inference context window for the session
- **Option D: Checkpoint warm-start** — use ThunderGate's checkpoint system (Principle 18) to pre-load Ghost Jon with a warm context state

Evaluate each option against:
- Context window cost (Haiku 4.5 is cheap but not free — Ghost Jon runs 241+ samples/day)
- Match rate improvement (will this actually fix the 0%?)
- Maintenance burden (when MEMORY.md updates, does Ghost Jon auto-pick it up?)
- Architectural fit with ThunderGate's design principles

### 2. The Match Rate Metric

The match rate is currently defined as binary: Ghost Jon's response either matches OpenClaw Jon's response or it doesn't.

But "match" is poorly defined. Evaluate:
- Is semantic similarity the right metric, not exact match?
- Should "HEARTBEAT_OK" vs "I'll check for HEARTBEAT.md... HEARTBEAT_OK" count as a match?
- What's the minimum match rate that means Ghost Jon is ready for cutover?
- Should match rate be weighted by message type (simple ack vs complex analysis)?

Recommend a better match rate definition.

### 3. Hermes Learning Loop Test Design

After Ghost Jon's context problem is fixed, we want to test the Hermes learning loop.

The learning architecture is documented at: `thundergate-research/HERMES_LEARNING_LOOP_ANALYSIS.md` in the main workspace (not this repo). Key points:
- Background review fork triggers after N turns/tool calls
- Fork reviews conversation and creates/updates skills or memory
- ThunderGate's learning module should live in `thundergate/src/learning/`
- Four files: `session_db.ts`, `skill_manager.ts`, `background_review.ts`, `review_prompts.ts`

Design a test that would prove the learning loop works:
- What test scenarios would trigger the background review fork?
- How do you verify that learning actually took place (not just that the code ran)?
- How do you verify that learning persists across sessions?
- What's the minimum "learning happened" bar for the 7-day gate?

### 4. The 7-Day Gate Criteria

Currently the gate is: 7 consecutive clean days + Doctor green = cutover ready.

But "clean day" is defined against a 0% match rate. That means every day is currently failing.

With the context fix:
- What's a realistic match rate target for "clean"? (80%? 90%? 95%?)
- Should the gate reset once context injection is deployed? (Ghost Jon Day 1 restarts)
- What does "Doctor green" mean in practice — what checks pass?

---

## Output

Write your analysis to `GHOST_JON_PRESSURE_TEST_ANALYSIS.md` in this directory.

Specifically recommend:
1. Which context injection option to implement (with reasoning)
2. Revised match rate definition and target threshold
3. Learning loop test design (3-5 concrete test scenarios)
4. Whether the 7-day gate clock should reset after the context fix

Update `ACTIVE_TASKS.md` when done.

---

## Rules
1. Read WHO_YOU_ARE.md, ARCHITECTURE.md first for background
2. Write to file — no terminal output
3. No code changes
4. No push
5. Hard truths. Don't sugarcoat the 0% match rate situation.
