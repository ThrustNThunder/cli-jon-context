# Learning Loop — Hermes Comparison + GJ Behavior Verification Brief
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Compare ThunderGate learning loop vs Hermes, implement gaps, verify Ghost Jon changes behavior after learning fires
**Mode:** WRITE CODE. Write files directly. No printing to terminal. No push.

---

## Background

ThunderGate has a working learning loop in `src/learning/triggers.ts` with 5 event-based triggers:
1. Task completes
2. Correction from Michael
3. Session ends
4. Failure occurs
5. Every 20 turns (backstop)

Hermes (the reference implementation) uses a different model — interval-based background fork after every ~10 turns or ~10 tool calls, spawning a full review agent with memory + skills toolset.

The question: **what is Hermes doing that ThunderGate isn't?** And critically: **does Ghost Jon actually change his behavior after learning fires?**

---

## The Hermes Trigger Model (reference)

From `thundergate-research/HERMES_LEARNING_LOOP_ANALYSIS.md`:

### Triggers
- `_skill_nudge_interval`: every 10 tool calls → reviews skills
- `_memory_nudge_interval`: every 10 turns → reviews memory
- Both run a **full background review fork** — separate AIAgent instance, bounded 16 iterations, silent (no user-visible output), memory+skills toolset only

### Background Fork
- Inherits full conversation history snapshot
- Appends review prompt as a synthetic user turn
- Bounded at 16 iterations
- Provenance tracked: `_memory_write_origin = "background_review"`

### Memory Review Prompt (key)
```
Review the conversation above and consider saving to memory if appropriate.
Focus on:
1. Has the user revealed things about themselves — their persona, desires, preferences, personal details?
2. Has the user expressed expectations about how you should behave, their work style, ways they want you to operate?
If something stands out, save it. If nothing, say 'Nothing to save.' and stop.
```

### Skill Review Prompt (key — bias toward update not create)
```
Review and update the skill library. Be ACTIVE — most sessions produce at least one skill update.
Preference order:
1. UPDATE A CURRENTLY-LOADED SKILL
2. UPDATE AN EXISTING UMBRELLA
3. ADD A SUPPORT FILE under an existing umbrella
4. CREATE A NEW CLASS-LEVEL SKILL (last resort)
```

### Skill Format
```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md
│   ├── references/
│   ├── templates/
│   └── scripts/
```

---

## ThunderGate vs Hermes — What to Compare

Read `src/learning/triggers.ts` carefully. Then compare against the Hermes model above.

**Specific gaps to investigate:**

### Gap 1 — Background Fork
Hermes spawns a **separate agent instance** for review. ThunderGate's `handleSessionEnd` / `handleBackstop` do regex pattern matching in-process. No actual LLM-based review. This means ThunderGate can only detect patterns it was hard-coded to look for.

**Question:** Should ThunderGate implement a proper background LLM review fork? Or is the event-based model with in-process extraction sufficient?

**Recommendation required:** If yes → design the fork. If no → explain why the current approach is adequate for our use case.

### Gap 2 — Skill Creation
ThunderGate creates `task_pattern` skills based on message count heuristics (`assistantMessages.length >= 3`). This is too crude — it creates skills on any session with 3+ assistant messages regardless of whether anything useful happened.

Hermes has a LLM-reviewed skill creation process with a strong bias toward *updating existing* skills rather than *creating new ones*. ThunderGate has no existing-skill-update logic at all.

**Task:** Implement a basic skill update path. When `handleTaskComplete` fires, check existing skills first. If a similar skill exists, update it. Only create new if nothing fits.

### Gap 3 — Review Prompts
ThunderGate has no LLM review prompts — it uses regex. Hermes has carefully crafted prompts that guide the review agent toward high-value captures.

**Task:** Write ThunderGate review prompts adapted from Hermes for our specific context (Jon + Michael + ThunderAI). Add to `src/learning/review_prompts.ts` (create if doesn't exist).

Memory review prompt should focus on:
- Michael's preferences and work style
- Corrections and adjustments to Jon's behavior
- Important facts about the team, business, projects

Skill review prompt should focus on:
- Technical procedures that worked
- Debugging paths that resolved issues
- Workflows that saved time

### Gap 4 — Behavior Change Verification
This is the most important one. After learning fires, **does Ghost Jon actually respond differently?**

Currently there's no test for this. T1-T3 verify that learning fires and persists. But they don't verify that the stored memories are ever *read back* and influence Ghost Jon's responses.

**Task:** Design and implement T6 — Behavior Change Test:
1. Fire a correction: "no jon, when I say 'the ship' I always mean the sailboat, not the RV"
2. Verify memory row exists (T1 already covers this)
3. NOW: inject a follow-up message: "Jon remind me what 'the ship' refers to"
4. Assert: Ghost Jon's response contains "sailboat" (not "RV", not "unclear")
5. This is the only test that proves the learning loop actually closes — extracted → stored → retrieved → behavior changed

This test requires Ghost Jon's system prompt to include a mechanism for retrieving recent memories at inference time. Check if `src/ghost/context.ts` currently loads from `session_db.memories` at all. If not, that's the missing link — the context assembler needs to pull recent memories and prepend them to the system prompt.

---

## Deliverables

### 1. Gap Analysis Report
Write `/tmp/cli-jon-context/LEARNING_LOOP_GAP_ANALYSIS.md`:
- For each of the 4 gaps above: current state, Hermes approach, recommendation, what you implemented
- Be honest about where ThunderGate is behind Hermes and where it's ahead

### 2. Code Changes
In `/home/ubuntu/thundergate-dev/src/learning/`:

**`triggers.ts`:**
- Add skill update path to `handleTaskComplete` — check existing skills before creating new
- Fix the failure trigger throttle issue flagged in T4 (either throttle or unique keys explicitly)
- Add basic review prompts call for `handleSessionEnd` (even if just structured extraction, not full fork)

**`review_prompts.ts`** (create):
- `MEMORY_REVIEW_PROMPT` — adapted from Hermes for ThunderAI/Michael context
- `SKILL_REVIEW_PROMPT` — adapted from Hermes, bias toward update not create

**`src/ghost/context.ts`:**
- After loading GJ context files, also query `session_db.getRecentMemories(10)` and prepend them to the system prompt under a `## Recent Memories` section
- This is the missing link that makes behavior change possible

### 3. T6 — Behavior Change Test
Add to `/home/ubuntu/thundergate-dev/src/ghost/learn-test.ts`:

**T6 — Behavior change after correction:**
1. Fire correction: "no jon, when I say 'the ship' I always mean the sailboat, not the RV"
2. Verify memory stored
3. Rebuild Ghost Jon context (includes recent memories now)
4. Inject message: "Jon remind me what 'the ship' refers to"
5. Call Ghost Jon's `shadowResponse` with the new context
6. Assert: response contains "sailboat"
7. Log: PASS or FAIL with the actual response shown

Updated test report should be:
```
Ghost Jon Learning Loop Test
=============================
T1 Correction burn-in:          PASS/FAIL
T2 No false positives:          PASS/FAIL
T3 Persistence on restart:      PASS/FAIL
T4 Failure idempotency:         PASS/FAIL
T6 Behavior change after learn: PASS/FAIL

Minimum gate bar (T1+T2+T3+T6): PASS/FAIL
```

T6 is now part of the minimum gate bar. Learning that doesn't change behavior isn't learning.

### 4. Build + Test
- `npm run build` clean
- `thundergate ghost learn-test` — all tests pass including T6
- Report results to `/tmp/learning-compare-complete.txt`

---

## Rules
1. Read cli-jon-context (WHO_YOU_ARE.md, ARCHITECTURE.md, prior analyses) first
2. Write files directly — no terminal output
3. No push
4. Update ACTIVE_TASKS.md when done
5. Hard truths in the gap analysis — if ThunderGate is behind Hermes in a meaningful way, say so
