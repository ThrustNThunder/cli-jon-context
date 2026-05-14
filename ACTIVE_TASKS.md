# Active Tasks — May 14, 2026

## 🟢 LOCKED IN TODAY (May 14, 2026)

### ThunderBrowser Phase 0 + Phase 1 scaffold landed in `thundergate-dev`
- Branch `thunderbrowser-phase01` carries a self-contained Phase 0+1
  scaffold under `thunderbrowser/` (kept off `thundercomm-stable` per
  the no-push-from-ThunderBase rule).
- **TB-0 Phase 0 plumbing**: MV3 manifest, background SW with
  alarm-driven heartbeat, WSS client with exponential-backoff reconnect
  + outbox replay, popup + options pages, dev-Chrome launcher script,
  IDB-backed audit store with chained SHA-256 prev_hash, fixture HTTP
  server on :7860, mock TG WSS server on :7861 with `.tbscript` runner,
  bridge module on :7862 with per-session command queue +
  ack/result tracking.
- **TB-1-1 content script bus + isolated-world injection** ✅ —
  declarative + programmatic injection, envelope-shaped CS↔SW bus,
  re-inject on stale ping, ref registry with WeakRef + lineage
  invalidation.
- **TB-1-2/1-3 snapshot_dom (structured) + query/get_text/get_url** ✅
  — interactive + landmark + text-bearing node retention, attr
  whitelist, accessible-name computation, 80 KB budget with truncation,
  ref handles, redactor-aware values.
- **TB-1-4 navigate + wait_for_load** ✅ — allowlist enforced, network-
  idle and DOMContentLoaded modes deferred to CS.
- **TB-1-5/1-6/1-7 click + fill + scroll_to** ✅ — visibility/clickable
  preconditions, scroll-into-view, native value-setter shim for
  React/Vue controlled inputs, precision-click stability re-check at
  250 ms.
- **TB-1-8..1-11 detect_modal / detect_error / detect_loading /
  is_logged_in** ✅ — heuristic modal classifier (cookie/auth/
  marketing/error/confirmation), error patterns (login_expired,
  http_5xx, access_denied), DOM-anchor login signal.
- **TB-1-13 allowlist** ✅ — hardcoded to `localhost:7860` +
  `*.aa.com`. Bridge has no command to expand it; this is the
  architectural lever against a compromised TG.
- **TB-1-14 input redactor** ✅ — runs before any CS→SW message;
  password / cc / ssn fields blanked, length preserved as `value_len`.
- **TB-1-15 state-pack format + AA pack** ✅ — versioned JSON ruleset
  with URL/DOM/text detectors and confidence weights; AA pack covers
  the 9 fixture states plus `password_change`, `timeout`,
  `captcha_blocked`. MutationObserver emits `state_detected` on
  coalesced changes (250 ms debounce). All 9 states resolve correctly
  in the offline state-pack test.
- **Fixture site** ✅ — `fixtures/aa/{login,dashboard,travel-planner,
  travel-planner_results,…_fare,…_passenger,…_payment,…_confirm,
  …_confirmed,timeout,password-expired,captcha}.html` served on
  `:7860`.
- **Tests green** (`npm test` in `thunderbrowser/`):
  - `tests/smoke.mjs` — envelope helper + state-pack shape + manifest
    references + bridge module imports.
  - `tests/state-pack.test.mjs` — 9 AA states resolved by URL+DOM+text
    detectors against synthetic fixtures.
  - `tests/bridge.e2e.mjs` — ws hello/ready handshake + command
    round-trip against the production-shaped bridge module.
- **Not wired (Phase 2+):** BYOAA scope tokens, device Ed25519 key +
  pairing flow, recording mode (`.tbrec`), gateway-side audit anchor
  in `context.db`, PBS state pack + fixtures, hot-pushed state packs,
  ThunderCommo confirmation gate.

## 🟢 LOCKED IN — May 13, 2026

### Prime Directive + Two-Mode Architecture
- **`docs/PROJECT_JON_PRIME_DIRECTIVE.md`** authored and locked in
  `thundergate-dev`. Michael's verbatim five-sentence directive sits at
  the top. Future Jon reads this file first on wake-up: who he is
  (Michael's Jarvis), what ThunderGate enables, the two-mode contract,
  full-autonomy goal, awareness substrate. Permanent record — append
  clarifications, never overwrite intent.
- **`docs/THUNDERGATE_DESIGN_PRINCIPLES.md`** mirrored into
  `thundergate-dev/docs/` (canonical, ships with the code) and extended
  with four new locked principles:
  - **#26 — TWO-MODE ARCHITECTURE (INFERENCE + CLOUD).** CLOUD is the
    always-available floor; LOCAL_INFERENCE is the destination. Switches
    flip `WorldState.processingMode`, log provenance, lose zero context.
  - **#27 — COMPLETE SITUATIONAL AWARENESS.** Four axes (user, infra,
    agent, world). Continuously maintained, not reactive. `WorldState`
    is the single substrate.
  - **#28 — AUTONOMOUS TASK EXECUTION.** Act, don't ask. Bounded only by
    explicit stop signals, actions that leave the system, and
    irreversible destructive ops.
  - **#29 — PROVENANCE FOR EVERY STATE CHANGE.** Append-only ledger,
    every mutation logs `(timestamp, actor, action, target, reason)`.
    No silent state mutations.

### Dual-Mode Architecture — Scaffold SHIPPED
- Basic scaffold landed in `thundergate-dev` (untracked → committed in
  the prime-directive run):
  - `src/world/state.ts` — `WorldState` + `ProcessingMode` enum
    (`CLOUD` / `LOCAL_INFERENCE`) + `LocalInferenceLiveness`. Single
    read-point for mode-aware branching.
  - `src/inference/local_provider.ts` — local-inference provider with
    health probe, mirrors liveness onto `WorldState`.
  - `src/provenance/ledger.ts` — provenance ledger primitive (Principle
    29 substrate). Append-only.
  - Runtime + CLI + doctor + ghost + config-loader wired through the
    new substrate (`src/core/runtime.ts`, `src/cli/main.ts`,
    `src/doctor/standalone.ts`, `src/ghost/harness.ts`,
    `src/config/loader.ts`).
- Cloud mode unchanged (Principle 26 floor preserved). Local-inference
  mode currently activates when the health probe finds an endpoint
  healthy; otherwise the runtime stays in `CLOUD`.

## 🟡 ACTIVE / NEXT (post-prime-directive)

### WorldState Expansion (per Principle 27 + Awareness Analysis §1, §5, §7.1)
- Today `WorldState` only carries `processingMode` and
  `localInference` liveness. Expand to the four-axis model:
  - **User state:** local time (Michael's TZ), active device,
    last-keyboard-touch per device, network class, tone trend from
    last-N inbound, posture override (`focus | pairing | afk`).
  - **Infra state:** disk on `~/.thundergate/` + `~/.openclaw/`
    partitions, SQLite WAL size + last `wal_checkpoint`, NTP drift,
    per-subsystem heartbeats (`ghost_last_log_at`,
    `learning_last_run_at`, `channel_<n>_last_traffic_at`,
    `relay_last_seen_at`), rate-limit headroom per provider.
  - **Agent state:** federation peer roster with `last_seen_at`,
    pending tasks per peer, current frame per channel.
  - **World state:** Anthropic / OpenAI / Voyage / xAI rolling health,
    queue depths per channel, API status-feed snapshots.
- Sampler runs on a 5–15 s cadence (per analysis §7.1). One in-memory
  object, read by `processMessage` before composing a turn and by
  Doctor for proactive triggers.

### Posture State Machine (per Awareness Analysis §5 + §7.6)
- Reads `WorldState` + Michael's manual overrides. Outputs three knobs:
  - **Response shape:** terse / full (driven by active device + tone
    trend).
  - **Default mode:** surface / deep (driven by current task class +
    explicit cues).
  - **Notification policy:** quiet / normal / urgent-only (driven by
    local time + posture override).
- Manual override commands: `tg posture focus | pairing | afk`. Sticky
  until cleared. Every transition writes provenance (Principle 29).
- Default-rules table + override audit trail land together.

### Proactive-Events Scheduler (per Awareness Analysis §3 + §7.5)
- Generalize the planned smart-compaction cron into a proactive-trigger
  registry. Each entry: `(condition predicate over WorldState, action
  callback, cooldown)`.
- Substrate for: ghost-drift early warning, stale-context warnings,
  end-of-day brief, pre-sleep notifier, dependency-status heartbeat,
  auto-research during dormancy window, promise tracker, calendar
  awareness (when wired).
- Channel registry already broadcasts outbound; scheduler just feeds
  it. Cooldowns prevent flapping.

### Ghost Jon 7-Day Clock
- Day 1 = May 10 (FK fix deployed). Need 7 consecutive clean days
  before cutover.
- May 11 pressure test passed clean (54 paired entries, 0 FK errors,
  median 705 ms).
- Monitor daily with `ghost status`. Promote when 7 consecutive clean
  days reached.
- Daily health check at 08:00 UTC flags err > 10 %, missing score
  rows, and any FK regression newer than the deploy.

### ThunderCommo Build 25 → 26+
- From Build 26 onward: Jon runs CLI Jon pressure test before Mack
  ships. Always. Locked workflow (#tnt May 10 13:46 ET). Build 27 +
  28 + 28b + 29 + 31 already shipped under the gate.

### thundercomm-stable Web UI Redesign
- Commit `fb62e6634a` sits on Mac side. Mack handles the push.

## ✅ DONE (recent rollup)

### May 13, 2026 — Prime Directive lock-in
- `PROJECT_JON_PRIME_DIRECTIVE.md` authored in `thundergate-dev/docs/`.
- `THUNDERGATE_DESIGN_PRINCIPLES.md` extended with Principles 26–29 and
  mirrored into `thundergate-dev/docs/` as the canonical version.
- Dual-mode scaffold (`src/world/`, `src/inference/`, `src/provenance/`
  + runtime/CLI/doctor wiring) committed to `thundergate-dev` master.

### May 11, 2026 — Build 31 (combined run)
- APNs iOS wiring (authorization, decline banner via
  `.notificationsDeclined`, device-token registration, foreground
  banner presentation, tap-to-channel routing via `.openChannel`).
- `subscribedChannels` loop iterating over `availableDirectAgents`
  (emits `direct:jon`, `direct:mack`, `direct:rex` plus `tnt` / `jmab`
  + active custom channel).
- Jon thinking-dots via Mack-format `typing: true` / `typing: false`
  events on `bridge.mjs` federation send, wrapped in try/catch so a
  relay hiccup never blocks dispatch.
- `MACK_MANUAL_STEPS_BUILD31.md` written to `apps/ios/` with Xcode-only
  steps + smoke test plan + conservative-choice notes.
- `xcodebuild Debug iphonesimulator` → **BUILD SUCCEEDED**, single
  pre-existing AppIntents warning unrelated.

### May 11, 2026 — earlier
- DM routing fix on iOS ThunderCommo (multi-channel auth on
  (re)connect; auto-route on incoming DM; re-auth on DM peer switch).
- Build 28 pressure test (source-level) — Section I (TNT watermark)
  CONDITIONAL PASS per brief verdict floor.
- Full workspace doc + repo audit — six superseded ThunderCommo design
  docs archived; four build briefs reorganized under `project_jon/`;
  `scripts/README.md` added.
- APNs iOS spec authored + enhanced with §3 Xcode config, §4a
  AppDelegate snippet, §4b in-app deny banner, §5a notification
  handling, §5b Bug #9 replay prevention, §9 provisioning &
  credentials.
- ThunderGate browser bridge implemented + wired (`src/channels/browser.ts`,
  port 9876, path `/browser`, per-peer command queue, audit ingestion).
- ThunderBrowser TB-1-1, TB-1-2 (snapshot + navigate), TB-1-3
  (read + write) landed in `extensions/thunderbrowser/`.
- Type-check + build clean.

### May 10 → May 11
- Ghost Jon pressure test PASSED (commit `5a5d2bf`).
- ThunderGate hardening — DB FK fix, `ghost status` display, systemd
  unit installed, ghost pairing timing, channel-conflict logging
  (commits `1a483c7`, `b167d6b`).
- Build 27 reports pushed (commit `0d19a51`).
- `thundermind_price_watch.py` rewritten with Newegg direct scrape,
  trusted-domain whitelist, median-of-3 decision price, loud Brave-key-
  missing warning in cron logs.

## 🔵 STILL TO DO

### Brave API Key Missing on ThunderBase
- `~/.openclaw/openclaw.json → tools.web.search.apiKey` is empty.
- Without it the price-watch Amazon / Micro Center fallback returns
  nothing. Newegg direct scrape covers the primary parts; key should
  be restored.

### Enable thundergate systemd Service
- Unit file exists but `systemctl status thundergate` shows inactive
  (dead). `sudo systemctl enable thundergate && sudo systemctl start
  thundergate` when ready.

### ThunderBrowser — Phase 1 remainders (post-May-14 scaffold)
- TB-0-7/0-8 device Ed25519 keypair + QR pairing flow + pinned-kid map.
- TB-1-12 audit chain: device signatures (placeholder today), gateway
  anchor table in `context.db`, server-ack-driven local rotation.
- TB-1-16 AA `.tbscript` happy-path + sold-out + password-expired +
  session-timeout + captcha scripts run end-to-end against the dev
  extension (today the happy-path script exists, the others are
  pending).
- TB-1-17 PBS state pack + fixtures.
- TB-1-18 recording mode (`.tbrec` dump + replay tool).
- Phase 1 exit gate (all 18 tickets green, both AA + PBS fixture suites
  in CI, audit chain integrity test, redactor + allowlist CI guards).

## Important Rules

- Never push to `thundercomm-stable` from ThunderBase — that's Mack's
  repo on Mac.
- No unsolicited changes to `openclaw.json` or gateway config.
- Pressure-test gate is non-negotiable for Build 26+.
- `<all_urls>` content-script matcher is the Phase 1 dev posture only —
  TB-1-13 (allowlist) replaces it before any production cut.
- **Prime Directive is permanent record.** Append clarifications;
  never overwrite Michael's words at the top of
  `PROJECT_JON_PRIME_DIRECTIVE.md`.
- **Principles are numbered and locked.** Renumbering is never
  allowed — Michael, Jon, and downstream docs cite by number.
- **Provenance is mandatory** (Principle 29). Every state mutation
  writes a row. PRs that mutate without logging fail review.
