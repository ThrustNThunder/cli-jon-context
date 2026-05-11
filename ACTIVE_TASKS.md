# Active Tasks — May 10, 2026 23:05 ET

## 🟡 ACTIVE / NEXT

### Ghost Jon 7-Day Clock
- Day 1 = May 10 (FK fix deployed tonight). Need 7 consecutive clean days before cutover.
- Today's err_rate still red (44.7%) — FK errors logged earlier today *before* the fix.
- Latest 3 log entries (after fix) are clean Haiku responses, no FK errors.
- Monitor daily with `ghost status`. Promote when 7 consecutive clean days reached.

### ThunderCommo Build 25 → 26
- Build 25 (Settings UX fixes) uploaded — grandfathered, last build without pressure-test gate.
- From Build 26 onward: Jon runs CLI Jon pressure test before Mack ships. Always.
- Pressure test gate is locked workflow (per #tnt May 10 13:46 ET).

### thundercomm-stable Web UI Redesign
- Commit fb62e6634a sits on Mac side. Mack handles the push.

## ✅ DONE TONIGHT (May 10)

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
- Falls back to current_est when no live data instead of NO_DATA.
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
