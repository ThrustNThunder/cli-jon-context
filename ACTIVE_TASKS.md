# Active Tasks — May 11, 2026 00:10 ET

## 🟡 ACTIVE / NEXT

### Ghost Jon 7-Day Clock
- Day 1 = May 10 (FK fix deployed). Need 7 consecutive clean days before cutover.
- May 10 still red (44.7% err) — FK errors logged earlier that day *before* the fix.
- **May 11 pressure test passed clean** (see Done Tonight). FK fix is holding;
  test ran 54 paired entries with **0 FK errors, 0 transport errors, median
  latency 705ms**.
- Open issue: today's `match_rate` after cleanup is **0% on 6 organic samples**.
  Not a FK regression — it's the Haiku ↔ OpenClaw voice gap. Needs real-traffic
  characterization, not a code fix.
- Monitor daily with `ghost status`. Promote when 7 consecutive clean days reached.
- Automated check wired into `~/.openclaw/workspace/scripts/daily-health-check.sh` (cron 08:00 UTC) — flags err > 10%, missing score rows, and any FK regressions newer than the deploy.

### ThunderCommo Build 25 → 26
- Build 25 (Settings UX fixes) uploaded — grandfathered, last build without pressure-test gate.
- From Build 26 onward: Jon runs CLI Jon pressure test before Mack ships. Always.
- Pressure test gate is locked workflow (per #tnt May 10 13:46 ET).

### thundercomm-stable Web UI Redesign
- Commit fb62e6634a sits on Mac side. Mack handles the push.

## ✅ DONE TONIGHT (May 10 → May 11)

### Ghost Jon Pressure Test (commit 5a5d2bf, May 11 00:08 ET)
- 50-message flood + edge cases + mid-run session-boundary test.
- **Verdict: PASSED.** 54/55 entries logged (the 1 "missing" was an
  empty-string user message correctly filtered upstream by `parseLine`).
- FK fix is holding — 0 new `FOREIGN KEY constraint failed` errors
  during or since the test.
- Latency: min 519 / median 705 / p90 4890 / max 5908 ms.
- Edge cases (5000-char, multi-script unicode, JSON+`<script>` payload,
  literal `"null"`) all logged correctly.
- Mid-run boundary file picked up on first 30s rescan, 5/5 pairs paired.
- Bug-fix bundled: pressure-test sessions (`ghost-test-*`) would
  otherwise pollute daily scoring with synthetic 0% match rows. Added
  scan-time and evaluator-time skip filters.
- Full report: `GHOST_PRESSURE_RESULTS.md` in thundergate-dev.

### ThunderGate Hardening (commits 1a483c7, b167d6b)
- DB foreign-key constraint fix — ghost entries no longer crash on DB write.
- `ghost status` display fixed — shows sessions_dir + file count.
- systemd unit installed (`/etc/systemd/system/thundergate.service`).
- Ghost pairing timing fixed — polls up to 30s for Haiku response before logging "not yet ready".
- Channel conflict logging improved — explains why port 8765 is skipped.
- dist/ recompiled and pushed so the runtime actually has the fixes.

### Build 27 Reports Pushed (commit 0d19a51)
- Gate + pressure-test reports + briefs landed on master.

### thundermind_price_watch.py Fixed
- Replaced Pangoly-fed snippet data with direct Newegg HTML scraping.
- Added trusted-domain whitelist (Newegg, Amazon, Micro Center, B&H, manufacturer stores).
- Added aggregator blacklist (Pangoly, Technobezz, camelcamelcamel, keepa, ebay, reddit, …).
- Median-of-3 decision price (less sensitive to single outliers).
- Falls back to current_est when no live data instead of NO_DATA (tagged `_(est)_` in Slack output so estimate-only rows are distinguishable from live prices).
- Loud warning surfaces in cron logs when Brave API key is missing (was silently returning NO_DATA).
- Test run on May 10 23:03 ET matches morning verification:
  GPU $9,349.99 · Mobo $1,290.99 · CPU $1,199.99 (BUY) · Phanteks $179.99 (BUY).

### Earlier Today
- MEMORY.md trimmed.
- GitHub PAT rotated.
- YouTube API token refreshed.
- ThunderCommo ack-on-receipt fix (Mack shipped).
- Admin dashboard updated.

## 🔵 STILL TO DO

### Brave API Key Missing on ThunderBase
- `~/.openclaw/openclaw.json → tools.web.search.apiKey` is empty.
- Without it the price-watch Amazon/Micro Center fallback returns nothing.
- Newegg direct scrape covers the primary parts, but key should be restored.

### Enable thundergate systemd Service
- Unit file exists but `systemctl status thundergate` shows inactive (dead).
- `sudo systemctl enable thundergate && sudo systemctl start thundergate` when ready.

## Important Rules
- Never push to thundercomm-stable from ThunderBase — that's Mack's repo on Mac.
- No unsolicited changes to openclaw.json or gateway config.
- Pressure-test gate is non-negotiable for Build 26+.
