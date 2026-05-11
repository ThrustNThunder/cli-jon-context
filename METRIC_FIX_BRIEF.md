# Ghost Jon Match Metric — Fix Brief
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Replace the broken Jaccard match metric with tiered semantic comparator + weighted daily score
**Mode:** WRITE CODE. Write files directly. No printing to terminal. No push.

---

## What's Broken

`src/ghost/evaluator.ts` and `src/ghost/harness.ts` use a simple Jaccard token-overlap metric with a 0.30 threshold and a binary daily pass/fail at match_rate ≥ 0.70.

Two concrete failure modes:
1. `HEARTBEAT_OK` vs `I'll check HEARTBEAT.md... HEARTBEAT_OK` → scores as no-match (diluted by preamble tokens). Should be a match.
2. Two 200-word responses meaning the same thing in different words → misses entirely. Should match.
3. Haiku 4.5 vs Sonnet — different models will never reach 70% token overlap even when semantically identical. Threshold is wrong for our setup.

---

## What to Implement

### 1. Tiered comparator in `src/ghost/harness.ts`

Replace `fuzzyMatch()` with a 3-tier system:

**Tier 1 — Exact/near-exact (free)**
- Normalize both strings: lowercase, collapse whitespace, strip punctuation
- If normalized strings are equal → score 1.0
- If Levenshtein distance / max_length ≤ 0.15 → score 1.0
- Catches: HEARTBEAT_OK, short acks, identical responses

**Tier 2 — Token Jaccard (cheap)**
- Same as current but: lower threshold to 0.20, remove the length ≥ 4 filter (it kills short meaningful tokens), use it as one signal not the verdict
- If Jaccard ≥ 0.20 → score 0.7
- If Jaccard ≥ 0.40 → score 0.85

**Tier 3 — Anthropic embeddings cosine (medium, ~$0.02/1M tokens)**
- Use Anthropic `voyage-3-lite` or whichever embedding endpoint is available on our key
- If embedding cosine ≥ 0.85 → score 1.0 (match)
- If embedding cosine ≥ 0.70 → score 0.5 (partial)
- If embedding cosine < 0.70 → score 0.0 (no match)
- Only call if Tier 1 + Tier 2 both returned < 0.85 (avoid unnecessary API calls)
- If embedding API unavailable/errors → fall back to Tier 2 score, log the fallback

**Final per-message score:** max of all tier scores. Binary match = score ≥ 0.75.

### 2. Weighted daily score in `src/ghost/evaluator.ts`

Replace mean of binary match_rate with:

```
day_score = Σ(weight_i × per_msg_score_i) / Σ(weight_i)
weight_i = log2(1 + max(openclaw_chars_i, thundergate_chars_i))
```

Short acks contribute little. Substantive responses dominate. This is honest.

**Updated status thresholds:**
```
green:  weighted_score ≥ 0.75  AND  error_rate < 0.05  AND  samples ≥ 10
yellow: weighted_score ≥ 0.60  AND  error_rate < 0.15  AND  samples ≥ 10
red:    anything worse, or samples < 10
```

Raise min samples from 5 to 10 — 5 was too noisy.

Keep `match_rate` in the output schema for backward compat but add `weighted_score` alongside it. Doctor reads `weighted_score` for gate decisions; `match_rate` stays for historical display.

### 3. Update GhostDailyScore interface

Add `weighted_score: number` to the interface. Keep all existing fields.

### 4. Update `ghost status` output

Show `weighted_score` in the status card alongside `match_rate`. Label it clearly:
```
  Match rate (binary):     23%   ← old metric, kept for reference
  Weighted score:          0.71  ← new metric, this is what counts
  Status:                  yellow
```

---

## Rules
1. Read cli-jon-context first
2. Write files directly — no terminal output
3. No push
4. `npm run build` must be clean
5. Update ACTIVE_TASKS.md when done
6. Write completion summary to `/tmp/metric-fix-complete.txt`
