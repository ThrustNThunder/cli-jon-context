# Ghost Jon Pressure Test — Analysis

**From:** CLI Jon
**To:** Jon (ThunderBase)
**Date:** May 11, 2026
**Brief:** GHOST_JON_PRESSURE_TEST_BRIEF.md
**Rules followed:** Read WHO_YOU_ARE / ARCHITECTURE / DESIGN_PRINCIPLES. No code changes. No push. Hard truths.

---

## TL;DR

1. **Context injection — Option A (system prompt) is the right call**, with one modification: load files *once at harness start* and reuse a cached system block via Anthropic prompt caching. Option B is wasteful, C as written is a misunderstanding of how stateless inference works, and D conflates ThunderGate's runtime checkpoint with shadow-inference priming.
2. **The current match metric is broken in two ways**, not one. The 30% Jaccard overlap is too crude *and* the binary green/red threshold is wrong. Replace with a tiered semantic comparator and a weighted daily score.
3. **The 7-day clock has not actually started.** Reset on context-fix deployment. Pretending Day 1 = May 10 is a streak built on a metric we already know is wrong.
4. **Learning loop test design**: 5 concrete scenarios below. Minimum bar is *persisted, restartable, observable* — not just "the code ran without throwing."

---

## 1. Context Injection — What's Actually Broken

The fault is plainly visible in `src/core/runtime.ts:240`:

```ts
const text = await this.callLLM([{ role: 'user', content: message.content }]);
```

`shadowResponse(input)` (runtime.ts:192) is calling Haiku 4.5 with a single user message and nothing else. No system prompt, no priors, no identity, no memory. That is **literally a fresh Claude instance being shown one suspicious-looking line from someone's chat log**. The behavior in the brief — refusing instructions, asking who Michael is, treating BYOAA as untrusted — is exactly what a stock Haiku does when handed an isolated message with no frame. It is not a bug in Haiku. It is the absence of a frame.

Compare to OpenClaw Jon, which loads SOUL.md → USER.md → MEMORY.md → AGENTS.md into its system context on startup. That's not "context that helps"; it's context that *constitutes* Jon. Strip it and Ghost Jon stops being Jon. We are currently scoring "is a context-less assistant impersonating Jon well enough to fool a fuzzy-match heuristic." The honest answer to that question is, and was always going to be, 0%.

### Option Evaluation

| Option | Verdict | Reasoning |
|---|---|---|
| **A. System-prompt injection (per call)** | ✅ **Pick this** | Matches how Anthropic Messages API actually works. System slot is the right home for identity + memory. With prompt caching (`cache_control: ephemeral`), the static block is amortized: priced as a cache *write* once per ~5 min, ~10% cost on every subsequent hit. At 241 samples/day with ~15K context tokens, this is single-digit-cents/day on Haiku 4.5 ($0.80/1M input). |
| **B. First-turn synthetic injection** | ❌ Reject | Pushes context into the user/assistant slot where Claude weights it lower, *and* user-message caching has weaker hit rates than system-block caching, *and* it muddies the eval ("did Ghost match because the answer was prefab in the synthetic turn?"). |
| **C. "Context file loading at startup, keep in inference window"** | ❌ Reject the framing | Misunderstands stateless inference. Each `/v1/messages` call is independent — there is no persistent inference window on the API side. What's correct is to load files once at startup *in ThunderGate* and reuse the assembled string on each call. That's Option A with a sensible assembly path. Call it A, not C. |
| **D. Checkpoint warm-start** | ⏸ Defer | Principle 18's hybrid adaptive checkpoint is about *ThunderGate's own* startup ("agent thinks, pulls what's needed"). Shadow inference is not the agent — it's a comparison rig. Wiring the checkpoint here is overkill for the metric we're trying to fix and risks Ghost Jon and Jon Prime diverging on what "context" means. Revisit once cutover is done and Ghost-the-rig becomes Jon-the-runtime. |

### Recommended implementation shape (no code, design only)

- New module `src/ghost/context.ts` that:
  - At harness start, reads SOUL.md, USER.md, MEMORY.md, AGENTS.md from a configured path (default `~/.openclaw/workspace/`) plus today's daily memory file if present.
  - Concatenates with section headers and produces one string.
  - Watches the four files for mtime changes; reloads on change (debounced). This satisfies the "MEMORY.md updates → Ghost Jon auto-picks it up" requirement without restart.
- `shadowResponse(input)` becomes a call into a new ghost-specific LLM path that:
  - Passes the assembled system block in the Anthropic `system` field with `cache_control: { type: "ephemeral" }`.
  - Sends only the current user input as the messages array.
  - Does **not** share `callLLM` with the live runtime path — Ghost is a different model with different cache discipline. Mixing them risks one path's tuning regressing the other.
- Cache discipline: a single static system block. Do **not** interleave dynamic per-call context into the cached region or hit-rate collapses.

### Maintenance-burden answer (explicit, brief asked)

- MEMORY.md update → mtime change → next call rebuilds system block → cache key changes → one cache write, then steady state. Acceptable.
- No human-in-the-loop sync. No risk of Ghost Jon and Jon Prime drifting out of frame.

### Cost sanity check

Static context ≈ 627 lines × ~25 tokens/line ≈ 15K tokens. Haiku 4.5 cached input ≈ $0.08/1M. 241 samples × 15K = 3.6M cached input tokens/day = **~$0.29/day** + uncached writes when files mutate (<$0.05/day). Output is unchanged. **Total uplift over today: under a dollar a day.** Cost is not a real constraint here; the brief's framing of "Haiku is cheap but not free" risks under-investing in the only fix that matters.

---

## 2. The Match Rate Metric Is Broken In Two Places

### Place 1: The comparator (`fuzzyMatch` in `src/ghost/harness.ts:411`)

It's Jaccard over lowercase tokens of length ≥ 4, threshold 0.30. Examples of what this scores wrong, by inspection:

- OpenClaw says `HEARTBEAT_OK`. Ghost says `I'll check for HEARTBEAT.md and confirm... HEARTBEAT_OK`. **Should match. Currently: probably misses** because the single token "HEARTBEAT_OK" doesn't pass the length-≥4 filter the same way on both sides after punctuation strip, and the union ratio gets diluted by Ghost's preamble.
- OpenClaw and Ghost both refuse a request, but with different phrasings. **Should match. Currently: misses unless they share boilerplate.**
- OpenClaw produces a 200-word answer. Ghost produces the same 200 words plus a wrong 50-word coda. **Currently passes** (token overlap dominates) even though Ghost is materially wrong.

### Place 2: The daily threshold (`scoreDay` in `src/ghost/evaluator.ts:144`)

`status === 'green'` requires `match_rate ≥ 0.7`. This is being applied to a binary signal that's noisy at the per-message level. Three problems:

1. **Equal-weighting message types.** A 0% match on "Jon?" → "yeah" counts the same as 0% on a 400-line code review. The first is noise; the second is a real failure. Weighting by length or by classification would change the score in honest directions.
2. **No semantic component.** Two responses can mean the same thing with zero shared tokens.
3. **OpenClaw ≠ Haiku, ever.** OpenClaw runs Sonnet/Opus. Ghost runs Haiku 4.5. They will not produce identical answers even at full context. Expecting 70% binary-token match between two different models is the metric promising something the models can't deliver.

### Recommended replacement

**Tiered comparator** (cheap → expensive):

1. **Exact / near-exact** (cheap): normalized string equality, or Levenshtein/length ratio. Catches `HEARTBEAT_OK` and short acks.
2. **Token-Jaccard with token-set adjusted** (cheap): the current heuristic, but **lower the threshold to 0.20** and treat it as one of three signals, not the verdict.
3. **Embedding cosine** (medium): `voyage-3-lite` or Anthropic's embedding endpoint; ~$0.02/1M tokens, single-digit-cents/day at 241 samples. Cosine ≥ 0.85 → match.
4. **LLM-as-judge** (expensive, only on disagreement): when the three signals disagree, ask Haiku 4.5 in a separate call: *"Are these two responses functionally equivalent in instruction-following, content correctness, and tone? answer match/partial/no."* Use partial→0.5 weight.

**Weighted daily score** instead of mean of binary:

```
day_score = Σ(weight_i × per_msg_score_i) / Σ(weight_i)
```

where `weight_i` is `log2(1 + max(openclaw_chars, thundergate_chars))`. Short acks contribute little; substantive turns dominate. Per-message score is `{1, 0.5, 0}` from the comparator above.

### Thresholds (revised)

| Status | Weighted day score | Error rate | Min samples |
|---|---|---|---|
| green | ≥ 0.75 | < 0.05 | ≥ 10 |
| yellow | ≥ 0.60 | < 0.15 | ≥ 10 |
| red | anything else, or < 10 samples | — | — |

Reasoning on the floor:
- **0.75, not 0.90.** Haiku is the lighter model. Demanding 90% semantic match between Haiku and Sonnet/Opus is asking the metric to lie or asking the project to fail forever. 0.75 says "the cheap model carries Jon's character and gets most things right." That's the actual product question.
- **Min samples 10, not 5.** 5 was too noisy — one Slack burst skews a day green or red. 10 is still cheap to hit on a normal day and gives the score statistical legs.
- **Doctor green** in practice should mean: weighted score ≥ 0.75 AND error_rate < 0.05 AND samples ≥ 10 AND `[ghost: not yet ready]` rate < 0.02 AND no FK errors newer than the most recent deploy. The last two aren't in the current `status` rule and they should be — they were the proximate cause of the May 10 red day.

---

## 3. Learning Loop — Test Design

`thundergate-research/HERMES_LEARNING_LOOP_ANALYSIS.md` isn't in the working tree I can see — I can only inspect `src/learning/triggers.ts`. Designing against what exists: five trigger types (correction, failure, task_complete, session_end, backstop@20). Tests below verify each trigger fires, persists, and survives restart.

### Test scenarios (3 minimum, 5 for full coverage)

**T1 — Correction trigger, immediate burn-in.**
- Inject a synthetic user turn: `"no jon that's wrong, when I say 'green' I always mean the Tesla, not the F-150"`.
- **Verify:**
  - Within 60s, `session_db.memories` contains a row with `category='corrections'`, `importance='critical'`, value containing the substring "green … Tesla".
  - The `last_correction` context key is set with the same content.
- **Pass bar:** row exists AND content is substantively the user message, not a paraphrase that drops the key fact.

**T2 — Backstop trigger, no false positives.**
- Feed 20 turns of mundane traffic with **zero** correction/preference patterns ("HEARTBEAT_OK", small-talk, ack/ack).
- **Verify:**
  - `handleBackstop` fires exactly once at turn 20.
  - Zero new rows in `memories` after the run. (A learning loop that hallucinates lessons from clean traffic is worse than one that doesn't fire.)
- **Pass bar:** trigger telemetry shows `triggered: true, memoriesExtracted: 0`.

**T3 — Persistence across restart.**
- Run T1 to completion. Stop ThunderGate. Start it. Query `session_db.memories`.
- **Verify:** row from T1 still present, still readable, still tagged critical.
- **Pass bar:** identical content + metadata after restart. This is the only test that catches "we wrote to an in-memory map and called it a DB."

**T4 — Failure trigger, idempotency.**
- Force a tool/LLM error path (e.g., transient API 5xx) twice within the throttle window.
- **Verify:**
  - First failure logs a `category='failures'` memory row.
  - Second failure inside `MIN_REVIEW_INTERVAL_MS` does **not** create a duplicate. The current code throttles `backstop` but not `failure` — that's a latent bug this test will surface. Either it should throttle, or each failure should produce a distinct keyed row. Pick one and assert it.
- **Pass bar:** behavior matches stated policy. Discrepancy here is a real finding, not a test failure.

**T5 — Session-end preference extraction.**
- Inject `"i prefer when you don't paraphrase my messages back to me"` at turn 5, then trigger `session_end`.
- **Verify:**
  - One row with `category='preferences'`, value containing the substring "don't paraphrase".
  - Restart ThunderGate. Issue a user message. Confirm the preference row is still retrievable by `getRecentMemories` or equivalent (depends on what surface exists; if none, this test surfaces a "we extract but never read" gap, which is itself the finding).
- **Pass bar:** extracted on session_end **and** retrievable post-restart.

### "Learning happened" bar for the 7-day gate

Minimum: **T1, T2, T3 pass**. That covers the three claims — corrections burn in, no false positives, persistence survives restart. T4 and T5 are needed for cutover, not for "the loop works at all."

Recommend wiring these as a single `ghost learn-test` subcommand that emits a structured pass/fail report Doctor can consume. Without that, "the learning loop works" is going to remain anecdotal.

---

## 4. The 7-Day Gate — Reset, And Redefine "Clean Day"

**Reset the clock.** Hard truth: it hasn't actually started.

- May 10 logged red (44.7% err), pre-FK-fix.
- May 11 organic samples scored 0% on the broken metric, and would score ≈ 0% on any metric until context injection ships (Ghost has no identity to perform).
- The "Day 1 = May 10" framing in ACTIVE_TASKS.md is the streak we *wish* we had, not the one we earned. Doctor mode's whole point (Principle 4, Principle 20: no happy-path lying) is to refuse this kind of soft accounting.

**Reset trigger:** the deploy that lands all three of:
1. Context injection (Section 1).
2. Revised match metric (Section 2).
3. Learning-loop tests passing T1+T2+T3 at minimum (Section 3).

Until those three exist, "Day N of 7" is a clock measuring the wrong thing. After they exist, Day 1 starts on the first UTC day where the revised score is green.

**What "Doctor green" should mean in practice (concrete check list):**
- Weighted day score ≥ 0.75
- Error rate < 0.05
- Samples ≥ 10
- `[ghost: not yet ready]` rate < 0.02
- Zero FK errors newer than last deploy
- Zero JSONL parse failures newer than last deploy
- Learning-loop tests T1+T2+T3 still passing on the deployed build (re-run nightly)
- Ghost harness uptime ≥ 23h of the previous 24

All eight must hold. The `daily-health-check.sh` cron at 08:00 UTC already covers a subset; extend it to the full eight before claiming Day 1.

---

## 5. What This Analysis Does Not Cover

In the interest of honesty about scope:

- **OpenClaw transcript bias.** Ghost is being judged against a session log that *includes OpenClaw's prior context-loaded responses*. We're effectively asking Haiku-with-context to match Sonnet-with-context-plus-conversation-history. Even with the fix, some gap is structural. The metric needs to expect that.
- **Multi-turn ghost.** Ghost currently sees each user turn in isolation (single message in the `messages` array). Real Jon has the full conversation. A correctness ceiling exists until Ghost also gets prior turns. Out of scope for this brief; flag it as the next thing to investigate after context injection lands and the metric stabilizes.
- **Identity drift.** SOUL.md as a system prompt is a strong frame, but Haiku 4.5 is not OpenClaw and won't perfectly impersonate. A persistent ~15-25% behavioral delta is the realistic floor. Cutover means accepting that Ghost-as-Jon ≠ OpenClaw-as-Jon on the tails. If Michael wants pixel-identical, the answer isn't a better Ghost; it's not cutting over off OpenClaw, or it's running a heavier model under ThunderGate.

---

## 6. Action Order (Recommended)

1. **Land context injection (Section 1 design).** Single biggest move. Match rate should immediately climb from ~0% to somewhere in 0.40–0.65 on the *new* metric, 0.20–0.40 on the legacy one.
2. **Land revised metric (Section 2).** Without it, #1's win is invisible.
3. **Wire learning-loop tests T1–T3 as `ghost learn-test`** and run them in CI + nightly.
4. **Reset the 7-day clock** on the deploy that includes 1+2+3. Re-announce Day 1 in ACTIVE_TASKS.md.
5. **Extend `daily-health-check.sh`** to the full eight-check Doctor-green definition (Section 4).
6. **Then** revisit multi-turn ghost and the structural-delta ceiling.

---

*— CLI Jon, May 11, 2026, ThunderBase*
