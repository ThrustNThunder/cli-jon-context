# ThunderBrowser — Phase 0-1 Build Tickets + PBS State Machine + ThunderCommo Narration Spec
**Author:** CLI Jon (ThunderBase)
**Date:** 2026-05-11
**Source of truth:** `THUNDERBROWSER_EXTENSION_DESIGN.md` (this repo)
**Scope:** Tickets actionable enough to pick up and start on Monday. No code, no push.

---

## 0. How to read these tickets

- IDs: `TB-<phase>-<n>`. Stable. Reference them in commits and PR titles.
- Owner hint: **CLI Jon** = ThunderBase side (bridge, scope service, mock TG, fixture server). **Mack** = Safari/iOS side; nothing in Phase 0-1 lands on Mack yet — his queue starts at Phase 5. **Ext Dev** = the engineer building the Chrome extension itself (could be CLI Jon if we don't hire; flagged separately so the work stays portable to a contractor without rewrite).
- Complexity: **S** ≈ ½ day. **M** ≈ 2-3 days. **L** ≈ 1 week. Honest, not aspirational. If a ticket grows past its bucket, split it before silently absorbing.
- A ticket isn't "done" until its acceptance criteria are mechanically demonstrable — log line, screenshot, fixture pass, audit-row read-back. "Looks right" doesn't count. Doctor-style audit is the floor.
- Phase 0 must be Doctor-green before any Phase 1 ticket starts. Phase 1 must be Doctor-green before BYOAA (Phase 2).
- Every ticket assumes Design Principle 1 ("don't break Michael's browser") and Principle 6 ("fail closed") apply. Where a ticket loosens this, it says so explicitly.

---

## 1. Phase 0 — Skeleton (target: 1 week)

The goal of Phase 0 is one demonstrable round trip: dev extension installs in a fresh Chrome profile, pairs to a mock ThunderGate, opens a WSS connection, survives a service-worker termination, and reconnects without losing a queued command. No DOM yet. No actions yet. Just plumbing — but plumbing that won't be rewritten in Phase 1.

### TB-0-1 — Manifest V3 scaffold
**Owner:** Ext Dev
**Complexity:** S

**Description.** Stand up the extension repo skeleton with a working `manifest.json` (MV3), background service worker stub, content script stub (no DOM logic yet — just a "hello" message responder), a popup HTML+JS shell, and an options page HTML+JS shell. Icons at 16/32/48/128. Versioned `1.0.0-dev0`.

The manifest must declare:
- `manifest_version: 3`
- `background.service_worker` pointing to the SW entry
- `permissions`: `storage`, `tabs`, `scripting`, `alarms`, `cookies` (the cookies permission is the read-presence-only path — see §7.4 of the design)
- `host_permissions`: empty in v0 (allowlist seeded only at pairing time per TB-0-8)
- `content_scripts`: declarative entries — empty in v0, populated by TB-1-1
- `web_accessible_resources`: none
- `content_security_policy.extension_pages`: `script-src 'self'; object-src 'self'`. No `'unsafe-eval'`, no `'unsafe-inline'`. (Design §7.7.)

**Acceptance.** `chrome://extensions` loads the unpacked extension with zero warnings/errors. Toolbar icon visible. Popup opens, shows a placeholder status pill. Options page opens at `chrome-extension://<id>/options.html`. SW reachable: clicking the popup logs a `runtime.sendMessage` round trip in DevTools.

**Dependencies.** None.

---

### TB-0-2 — IndexedDB schema + storage layer
**Owner:** Ext Dev
**Complexity:** M

**Description.** Define and create the IndexedDB schema the rest of Phase 0-1 will write to. Single database `thunderbrowser`. Object stores:

- `keypair` — `{ id: "device", publicKeyJwk, createdAt }`. Private key handle is non-extractable (per design §7.1) and lives only as a `CryptoKey`, never serialized; this store records the public side for forensic inspection and the metadata.
- `pairing` — `{ id: "current", extensionPairId, paired_at, tg_kid_pubkeys: [{kid, alg, pubkeyJwk, signed_by?}], bundle_hash }`.
- `scope` — current and previous scope tokens, raw JWT + decoded payload + `actions_used`. Capped at 10 rows.
- `runs` — `{ run_id, scope_id, label, started_at, ended_at?, state, expected_actions?, actions_dispatched, last_action_id_acked }`.
- `audit` — `{ action_id, run_id, ...action record from design §4.4..., prev_hash, signature, server_acked: bool }`. Indexed by `run_id`, by `server_acked=false`, by `ts_completed`.
- `state_pack` — versioned state-detector ruleset blobs (used in Phase 1 by TB-1-15).
- `settings` — allowlist additions, redactor regex denylist, kill-switch flag, last reconnect attempt timestamps.

A small `storage.ts` module wraps the schema and exposes typed `get`/`put`/`delete`/`scan` helpers. `chrome.storage.session` is used for hot, ephemeral state (current run id, scope-in-flight) so SW restarts don't lose it within a session.

**Acceptance.** Unit test (in `tests/storage.spec.ts`, run with `vitest`) round-trips a write/read across every store. A second test simulates SW restart by closing and reopening the IndexedDB connection and verifies a written `audit` row is still there. Schema upgrade path (`onupgradeneeded`) writes the version delta to the SW console so future migrations are debuggable.

**Dependencies.** TB-0-1.

---

### TB-0-3 — `platform.ts` shim + webextension-polyfill vendored
**Owner:** Ext Dev
**Complexity:** S

**Description.** Vendor `webextension-polyfill` at a pinned hash inside `vendor/webextension-polyfill/`. Build a `src/platform.ts` module that re-exports `runtime`, `tabs`, `scripting`, `alarms`, `storage`, `cookies`, and `windows` from either `chrome.*` (Chrome build) or `browser.*` (Safari build) based on a `process.env.PLATFORM` token resolved at build time. All other source files import only from `./platform`, never from `chrome.*` directly. (Design §6.2 — this is the Safari-conversion lever.)

**Acceptance.** Grep for `chrome\.` in `src/` returns zero hits outside `platform.ts` and the manifest tooling. A 30-second smoke build with `PLATFORM=safari` (skipping signing) emits a bundle that loads cleanly in the `webextension-polyfill` test harness; no runtime errors on import.

**Dependencies.** TB-0-1.

---

### TB-0-4 — Background SW + alarm-driven heartbeat
**Owner:** Ext Dev
**Complexity:** M

**Description.** The SW entry point installs a `chrome.alarms` alarm `tb_heartbeat` at 25s period. The alarm handler:

1. Checks WSS state; if `OPEN`, sends a `ping` event with monotonic seq; if `CLOSED`, calls into TB-0-5's reconnect logic.
2. Reads `last_action_id_acked` from `chrome.storage.session`; if stale (>2 min) re-reads from IndexedDB to recover from SW eviction.
3. Logs heartbeat outcome to a ring buffer (last 200) accessible from the options page audit viewer (Phase 1 wires the UI; the buffer exists here).

`setInterval` and `setTimeout > 30s` are banned in SW code — they die with the worker. The lint rule is added in TB-0-1's tsconfig/eslint.

**Acceptance.** Manually evict the SW via `chrome://serviceworker-internals` "Stop" and observe a heartbeat within ≤25s of the next event-driven wake (e.g., popup click). The ring buffer shows continuous heartbeats with no >30s gaps under normal load. A test alarm fires correctly with the dev extension reloaded.

**Dependencies.** TB-0-1, TB-0-3.

---

### TB-0-5 — WSS client: connect / disconnect / reconnect / queue replay
**Owner:** Ext Dev
**Complexity:** L

**Description.** Implement the SW-side WSS client per design §2.2:

- Opens `wss://<gateway>/browser` with subprotocol `thunderbrowser.v1` and `Authorization: Bearer <jwt>` header (JWT signed by the device key — see TB-0-7).
- On `open`: sends `ready` event with `{ ua, chrome_version, allowlist, active_tabs, bundle_hash }` (bundle hash from TB-0-1 build metadata).
- On `hello` from server: starts dispatch loop.
- On WSS `close`/`error`: exponential backoff (250 ms → 30 s, factor 2, full jitter), with an immediate retry on SW wake if `last_close > 5 s ago`.
- On reconnect: sends `resume` event carrying `last_command_id_acked` so the bridge replays queued commands.
- Persists `last_command_id_acked` to IndexedDB inside the same transaction that writes the corresponding `result`/`error` upstream commit, so a crash between ack and persist replays at most one command — never zero (no silent loss).
- Heartbeat hook from TB-0-4 feeds in here.

Envelope validation is strict: any inbound message missing `v`, `id`, `ts`, or `type` is rejected with `error { code: "BAD_ENVELOPE", retriable: false }`. Per design §2.4, unknown required fields → reject. Unknown optional fields → ignored.

A `wssClient.dispatch(cmd)` method is exposed for later tickets but its body is a TODO that just logs in Phase 0.

**Acceptance.** Run against `tg-mock` from TB-0-10. Verify:
1. Connect + `hello`/`ready` exchange logged.
2. Heartbeat round-trips for 5 minutes.
3. Killing the mock server triggers backoff with the documented schedule (capture timestamps).
4. Restarting the mock after 30 s of downtime: client reconnects, replays a queued command issued during the outage, and the SW emits a single `result` (not duplicates).
5. Forced SW termination mid-flight: on next wake, the client reconnects and the in-flight command's `result` is emitted at most once.

**Dependencies.** TB-0-2 (`audit`+`runs` stores for resume bookkeeping), TB-0-4 (alarm wakes drive reconnect), TB-0-7 (JWT signer for the auth header).

---

### TB-0-6 — ThunderGate Browser Bridge (`src/browser/bridge.ts`)
**Owner:** CLI Jon
**Complexity:** L

**Description.** New module on the ThunderGate side. Hosts the WSS endpoint `/browser`. Per-extension session state: keyed by `extension_pair_id`. Each session owns:

- A bounded command queue (default 64 entries; backpressure via `error { code: "QUEUE_FULL" }` to upstream callers — Jon's reasoning loop, not the extension).
- An in-flight map `command_id → { dispatched_at, scope_id, expecting: "ack"|"result" }` with timeouts: 2 s ack → reissue; 30 s result → mark `TIMEOUT`.
- Audit ingest: writes the `audit` event entries into a new `action_audit` table in `context.db` (single source per Design Principle 1) and updates `audit_anchors` every 60 s by re-signing the chain head with the gateway key.
- Pinned device pubkey check on every connect; rejects JWT mismatch with HTTP 401 before WSS upgrade completes.

Subprotocol negotiation must accept exactly `thunderbrowser.v1`; anything else returns 426 Upgrade Required.

Bridge exposes a small in-process API for Jon's reasoning loop:
```
sendCommand(extensionPairId, scopeId, action, params) → Promise<Result>
streamEvents(extensionPairId) → AsyncIterable<Event>
listSessions() → SessionInfo[]
```
Jon's reasoning loop is *not* part of this ticket. The API surface is just the bridge contract.

The module owns no business logic about AA, PBS, or scope policy. It enforces transport invariants only.

**Acceptance.** Two new tables in `context.db` (`action_audit`, `audit_anchors`) with migrations checked in. `tg-mock` (TB-0-10) can be replaced by the real bridge for an end-to-end Phase 0 demo: dev extension pairs, connects, heartbeats, and the bridge records nothing in audit (no actions yet) while keeping session state across a forced disconnect. Bridge unit tests cover: bad JWT (401), bad subprotocol (426), queue overflow (`QUEUE_FULL`), ack-timeout reissue, audit chain anchor written on 60 s schedule.

**Dependencies.** TB-0-7 (server side must know the pinned device pubkey format), TB-0-8 (pairing writes the pubkey).

---

### TB-0-7 — Device keypair generation + non-extractable storage
**Owner:** Ext Dev
**Complexity:** M

**Description.** On first run (options page button "Generate device key" — gated behind a confirm dialog because regenerating revokes pairing):

```
const kp = await crypto.subtle.generateKey(
  { name: "Ed25519" },
  false,                            // extractable: false — this is the point
  ["sign", "verify"]
);
```

Persist the `CryptoKey` handle to IndexedDB (`keypair` store from TB-0-2). The public key is exported to JWK and stored alongside; the private key is *never* exported. JWT signing helper `signJwt(payload, ttlSec)` produces a `EdDSA`-signed JWT with the device key for the 5-minute connection token used in TB-0-5.

90-day rotation: implemented in a deliberately separate Phase 2+ ticket (TB-2-X to be filed). v0-1 just generates once and rotates by re-pairing.

**Acceptance.** Generated key shows in DevTools IndexedDB as an opaque `CryptoKey` with `extractable: false`. `signJwt({}, 60)` produces a verifiable JWT against the exported public key. Manually re-running `generateKey` and asserting old keys can no longer sign by the same handle: handled by clearing the `keypair` store first; a guard in the options page prevents accidental overwrite. Memory test: kill the SW, re-open, key handle still loadable (IndexedDB persisted), JWT still signable.

**Dependencies.** TB-0-2.

---

### TB-0-8 — Pairing flow: QR + pair record + pinned TG kid map
**Owner:** Ext Dev + CLI Jon
**Complexity:** L

**Description.** End-to-end pairing per design §7.1 and §4.1.

Extension side (Ext Dev):
1. Options page button "Pair with ThunderGate" → calls into TB-0-7 to ensure device key exists.
2. Generates a one-time pairing code (16 bytes urlsafe-base64) and stores it with a 5-minute TTL.
3. Renders a QR encoding `{ pair_code, device_pubkey_jwk, extension_pair_id, version, gateway_hint }`. Use a vendored QR lib at pinned hash; do not call any external QR service.
4. Polls `/browser/pair-status?code=<pair_code>` over HTTPS (not WSS) every 2 s for up to 5 min; receives `{ paired: true, tg_kid_pubkeys: [...], allowlist_seed: ["aa.com","*.aa.com","<pbs domain>","*.<pbs domain>"] }`.
5. Writes pinned kid→pubkey map to IndexedDB and seeds the allowlist. Sets pairing as complete; subsequent SW starts use it.

ThunderGate side (CLI Jon):
1. New endpoint `POST /browser/pair-init` — Michael's ThunderCommo posts `{ pair_code, device_pubkey_jwk, extension_pair_id }` after scanning the QR.
2. ThunderGate writes pairing record into `context.db` (`browser_pairings` table) with `paired_at`, `user_id`, `bundle_hash_expected`.
3. `GET /browser/pair-status` returns the kid→pubkey set and allowlist seed once paired.
4. `DELETE /browser/pair` revokes (used by Phase 2+).

The PBS domain in the allowlist seed is config-driven on the TG side — written as `<airline_pbs_domain>` placeholder for the AA pilot use case Michael described. Initial value: TBD; flagged in `THUNDERBROWSER_OPEN_QUESTIONS.md` if not known at build time.

**Acceptance.** End-to-end: fresh extension in a fresh profile → pair QR rendered → ThunderCommo simulator (or a curl script standing in for it) posts `pair-init` → extension status flips to "paired" within one poll cycle → reconnect uses the pinned kid map. Negative test: a `pair-init` with a wrong `device_pubkey_jwk` (mismatched against the QR) is rejected with 400; nothing is written.

**Dependencies.** TB-0-2, TB-0-6, TB-0-7.

---

### TB-0-9 — Dev Chrome profile + launcher script
**Owner:** Ext Dev
**Complexity:** S

**Description.** Per design §8.2. Script `scripts/dev-chrome.sh` (and `.ps1` for Windows just in case Mack runs on Parallels) that launches a separate Chrome profile at `~/Library/Application Support/Google/Chrome ThunderDev/` (mac) or platform equivalent, with `--load-extension=dist/chrome` and `--no-first-run`. The dev extension's manifest sets `name: "ThunderBrowser (dev)"`, yellow popup background, distinct dev icon. Cross-talk guard: the dev SW refuses to connect to a `wss://gateway.*` host (anything not `gateway-dev`); the bridge from TB-0-6 refuses dev-build bundle hashes when running in prod mode. Both checks log a hard failure with a clear message; no silent fallback.

**Acceptance.** Running `dev-chrome.sh` opens a Chrome window with cookies/history empty (separate profile confirmed), the yellow popup loads. Forcing the dev SW to connect to a prod-looking URL produces the hard failure. Forcing a prod build to connect to dev TG fails the same way.

**Dependencies.** TB-0-1.

---

### TB-0-10 — `tg-mock` server + first `.tbscript`
**Owner:** CLI Jon
**Complexity:** M

**Description.** Per design §8.4. A Node script `scripts/tg-mock.ts` runs a WSS server on `wss://localhost:7861/browser` (self-signed cert auto-generated on first run, instructions in README). It accepts the same handshake as the real bridge but with relaxed JWT verification (any well-formed JWT signed by a pinned set of dev keys passes). It loads a `.tbscript` file from disk and walks the commands sequentially against the connected extension, asserting on event return values.

`.tbscript` format (line-oriented, comment-friendly):
```
# scripts/phase0_smoke.tbscript
EXPECT_READY
EXPECT_HEARTBEAT within 30s
SEND_EVENT { kind: "info", body: { msg: "Phase 0 mock alive" } }
EXPECT_HEARTBEAT within 30s
ASSERT no errors
```

Phase 0 ships exactly one script: `phase0_smoke.tbscript` (the lines above). Phase 1 will add the AA fixture scripts.

The runner exits 0 on pass, non-zero on first failure. Output is one line per directive plus a summary block at the end. Designed for CI.

**Acceptance.** `tg-mock` with `phase0_smoke.tbscript` against the dev extension (TB-0-9) exits 0 within 90 s, and the extension's audit ring buffer shows two heartbeats. Stopping the mock between heartbeats and restarting it: runner picks up where it left off (resume from `last_command_id_acked`), exit 0.

**Dependencies.** TB-0-5, TB-0-6 (must share the same envelope code; mock and real bridge import a common `protocol.ts`).

---

### TB-0-11 — Fixture server scaffold (Express on :7860)
**Owner:** CLI Jon
**Complexity:** S

**Description.** Stand up the fixture server skeleton from design §8.3. Express on `http://localhost:7860`. Phase 0 ships only the bones: a single `GET /` returning a "ThunderBrowser fixture server" health page; a static directory layout `fixtures/aa/` and `fixtures/pbs/` ready for Phase 1 content; a `--scenario` flag that picks a subdirectory to serve. No AA or PBS pages yet.

The server logs every request to stdout with timestamp + path; Phase 1 will use this for fixture-pass attribution.

**Acceptance.** `npm run fixture-server -- --scenario aa-empty` starts the server. `curl localhost:7860/` returns 200 with the health page. Subdirectory routing works (404 on `/login` is correct in Phase 0; Phase 1 fills it).

**Dependencies.** None (independent stream).

---

### Phase 0 exit gate

All of:
1. TB-0-1 through TB-0-11 marked done by acceptance criteria.
2. End-to-end Phase 0 demo recorded (screen capture, or a written walkthrough with terminal output): fresh profile → install → pair → connect → SW terminated → reconnect with queue replay → `phase0_smoke.tbscript` exits 0.
3. Doctor check added to `daily-health-check.sh`: `tg-mock` smoke pass within the last 24 h.
4. No `chrome.*` references outside `platform.ts` (re-check the lint rule).
5. No `console.error` calls during the demo. Errors are routed through the structured logger.

If any item is yellow, Phase 1 does not start.

---

## 2. Phase 1 — Core Actions (target: weeks 2-3)

Phase 1 makes the extension act on a page. The cut line is intentionally narrow: read + a small clickable/fillable surface + page-state detection + redaction + audit + allowlist enforcement + AA *and* PBS fixture pages green. No BYOAA scope tokens yet (that's Phase 2) — Phase 1 runs under a hardcoded `dev_scope` that whitelists everything on `localhost:7860` and the AA/PBS fixture origins, so the action surface can be exercised before the authority service lands. The dev scope is *only* honored by the dev build (TB-0-9 guard); the prod build refuses it.

### TB-1-1 — Content script bus + envelope + isolated-world injection
**Owner:** Ext Dev
**Complexity:** M

**Description.** Declarative content script entry per design §1.3 — manifest `content_scripts` with `matches` derived from the allowlist (initially the fixture origins + `*.aa.com` + the PBS origin from TB-0-8). `run_at: "document_start"` so the CS can observe interstitials. `world: "ISOLATED"` — never `MAIN`. Programmatic injection helper `injectIfMissing(tabId)` for ad-hoc tabs (rarely used; default is declarative).

Message bus: same envelope as WSS (`{v,id,ts,type,scope,ref,body}`). CS ↔ SW transport via `chrome.tabs.sendMessage` (SW→CS) and `chrome.runtime.sendMessage` (CS→SW). The SW maintains `tabId → { csReady: bool, lastPing: ts }`; before dispatching, it pings the CS with a 500 ms timeout and re-injects on failure (per design §2.3).

A `ref` registry per tab maps opaque handles (`el#12`) → `WeakRef<Element>` plus a hash of the element's lineage. References expire on navigation; lookup of an expired ref returns `error { code: "ELEMENT_NOT_FOUND" }`.

**Acceptance.** Round-trip a no-op command (`SW→CS→ack→result`) on the fixture server. Pull the plug on a tab (close it mid-command) and verify the SW emits `cs_disconnected` and the original command resolves to `error { code: "TAB_CLOSED", retriable: false }`. Re-inject test: kill the CS via DevTools "Stop", send a command, observe re-injection within 1 s.

**Dependencies.** TB-0-1, TB-0-5, TB-0-11.

---

### TB-1-2 — `snapshot_dom` (structured mode)
**Owner:** Ext Dev
**Complexity:** L

**Description.** Per design §3.2. `structured` mode is the default and the only one shipping in Phase 1; accessibility and text modes are deferred to Phase 2.

Algorithm (in the CS, after the redactor from TB-1-14 runs):
1. Traverse `document.documentElement`. Keep nodes that are interactive (`a`, `button`, `input`, `select`, `textarea`, `[role]`, `[tabindex]`, `[onclick]`), landmark (`main`, `nav`, `header`, `footer`, `aside`, `[role=banner|navigation|main|complementary]`), or contain trimmed text.
2. For each kept node, emit `{ ref, tag, role, accessible_name, text (≤2KB), attrs (whitelisted), bbox, children }`. Whitelisted attrs: `id`, `name`, `type`, `value` (post-redactor), `href`, `aria-*`, `data-testid`, `data-aa-*`, `placeholder`, `disabled`, `checked`, `readonly`, `required`.
3. Compute a stable hash `sha256(canonical(tree))` for the audit record.
4. Cap output at 80 KB; if exceeded, switch to a degraded mode that drops text > 200 chars and emits a `truncated: true` field.

Performance budget: ≤150 ms on a 5,000-node page (median consumer laptop). If exceeded on a fixture page, ticket reopens with a profiling task.

**Acceptance.** Fixture page `/aa/dashboard` (from TB-1-15) returns a snapshot in <80 KB containing the AAdvantage anchor, the Travel Planner nav link, and at most 200 nodes. Hash is deterministic across two consecutive snapshots of an unchanged page. Inputs with `type=password` and `autocomplete=cc-*` show `[REDACTED:password]` / `[REDACTED:cc]` per the redactor.

**Dependencies.** TB-1-1, TB-1-14, TB-1-15.

---

### TB-1-3 — `query` / `get_text` / `get_url`
**Owner:** Ext Dev
**Complexity:** M

**Description.** Per design §3.2.

- `query({ selector?, text?, role?, name?, limit? })` — runs the matchers in the CS, returns `matches: [{ ref, tag, role, text, bbox, attrs }]`. `text` matcher uses case-insensitive substring; `role`+`name` match aria role + accessible name (preferred over CSS for resilience). At least one of `selector`, `text`, or `role`+`name` is required; missing all returns `error { code: "BAD_PARAMS", retriable: false }`. `limit` default 20, max 200.
- `get_text({ ref })` — returns the element's `textContent` truncated to 8 KB.
- `get_url({ tab_id })` — returns `{ url, title }` from the tab the CS lives in (CS reports; SW does not read `tab.url` directly because the URL surface is treated as DOM evidence, not browser-API evidence).

**Acceptance.** Against `/aa/search-results`, `query({ role: "button", name: "Select" })` returns the three "Select fare" buttons in order. `get_text({ ref })` on the first one returns its label text. `get_url` returns the search-results URL.

**Dependencies.** TB-1-1, TB-1-2 (shares the `ref` registry).

---

### TB-1-4 — `navigate` + `wait_for_load`
**Owner:** Ext Dev
**Complexity:** M

**Description.** Per design §3.1.

- `navigate({ url, tab_id?, new_tab?, wait_for, timeout_ms })` — SW resolves `tab_id` (creates new tab if `new_tab`), enforces allowlist (TB-1-13), calls `chrome.tabs.update` with the URL, then defers to `wait_for_load`. Errors: `NOT_ALLOWLISTED` (pre-flight), `NET_ERROR` (frame error), `TIMEOUT`.
- `wait_for_load({ tab_id, condition, timeout_ms })` — three modes:
  - `load`: `chrome.tabs.onUpdated` status === "complete".
  - `domcontentloaded`: CS reports `readyState === "interactive"`.
  - `network_idle`: CS observes 500 ms of no new fetch/XHR starts (uses a small wrapper around `PerformanceObserver` for resource entries). 5 s hard cap inside the timeout window.

Long-running waits emit `progress` events at 1 Hz so Jon's reasoning loop can decide to abort.

**Acceptance.** `navigate` to `/aa/login` with `wait_for: "load"` returns `{ final_url, status_code: 200 }`. With `wait_for: "network_idle"` on a fixture page that triggers a 2 s XHR storm, the action waits until the storm stops + 500 ms. Timeout test: navigate with 1 s timeout to a fixture page that sleeps 3 s → `TIMEOUT` with `progress` events captured.

**Dependencies.** TB-1-1, TB-1-13.

---

### TB-1-5 — `click`
**Owner:** Ext Dev
**Complexity:** M

**Description.** Per design §3.3. The CS:
1. Resolves the `ref`; emits `ELEMENT_NOT_FOUND` if missing or stale.
2. Verifies clickability: visible (`bbox` non-zero, not `display:none`/`visibility:hidden`), not `disabled`, not `pointer-events:none`, not occluded (sample center pixel via `document.elementFromPoint`). Emits `NOT_CLICKABLE` or `OUT_OF_VIEW` otherwise.
3. Scrolls into view if needed (`scrollIntoView({block: "center"})`).
4. Dispatches `pointerdown`/`mousedown`/`pointerup`/`mouseup`/`click` synchronously, with `bubbles: true`, `cancelable: true`. Records `dom_before_hash` from the most recent snapshot (or computes one on-the-fly if stale).
5. If `expect_navigation`, awaits `chrome.webNavigation.onCommitted` for the tab for up to `timeout_ms` (default 10 s); returns `{ navigated, final_url }`.
6. Emits `dom_after_hash`.

A "precision click" mode for irrevocable buttons (used in TB-1-15's AA-fixture confirm test and in the PBS save-ballot flow): performs a stability check by re-querying with the same matcher 250 ms apart and confirming the same `ref` returns. If not stable, returns `error { code: "ELEMENT_UNSTABLE", retriable: true }`. The precision flag is on the *server* side of the protocol (the bridge sends `precision: true`), so the CS always honors it when present.

**Acceptance.** On `/aa/dashboard`, clicking the Travel Planner nav (by `ref` from `query`) navigates to `/aa/travel-planner` with `expect_navigation: true`. On a fixture page with a `display:none` button, click returns `NOT_CLICKABLE`. Precision click against the fixture confirm button (TB-1-15) succeeds with stability check passing in <500 ms.

**Dependencies.** TB-1-1, TB-1-3, TB-1-12 (audit hashes), TB-1-15.

---

### TB-1-6 — `fill` (non-secret)
**Owner:** Ext Dev
**Complexity:** M

**Description.** Per design §3.3, secret path deferred to Phase 2.

CS resolves the `ref`, asserts it's an `input` or `textarea`, not `readonly`, not `disabled`. Sets `.value` and dispatches `input` and `change` events with `bubbles: true`. For React/Vue controlled inputs, uses the native value setter shim (`Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set.call(el, value)`) before the events — without this, React resets to the old value on the next render. This is well-known but easy to miss; the ticket calls it out explicitly because losing 30 minutes debugging a controlled-input regression isn't a good Tuesday.

Returns `{ ok: true, value_hash: sha256(value) }`. The plaintext value never appears in any log or audit record — only the hash.

Errors: `NOT_FILLABLE` (not an input), `READONLY`.

**Acceptance.** On `/aa/passenger`, filling the "first name" field with "Michael" updates the React state (visible in the DOM and triggers the downstream validation). `value_hash` matches `sha256("Michael")`. Attempting to fill the AAdvantage number field marked `readonly` returns `READONLY`. The plaintext value is grep-absent from the SW logs.

**Dependencies.** TB-1-1, TB-1-3, TB-1-14.

---

### TB-1-7 — `scroll_to`
**Owner:** Ext Dev
**Complexity:** S

**Description.** Per design §3.3. Two modes:
- `{ ref }` → `scrollIntoView({block: "center", behavior: "instant"})`.
- `{ x, y }` → `window.scrollTo(x, y)`.

Returns `{ scroll_pos: { x, y } }` from `window.scrollX`/`scrollY` after the scroll.

**Acceptance.** On a long fixture page, scrolling to a `ref` near the bottom brings it into the center of the viewport. `{ x: 0, y: 0 }` returns to top.

**Dependencies.** TB-1-1, TB-1-3.

---

### TB-1-8 — `detect_modal`
**Owner:** Ext Dev
**Complexity:** M

**Description.** Per design §3.4. Modal heuristic in the CS:

1. Scan elements with `position: fixed` *or* `position: absolute` *and* `z-index >= 1000` *and* `bbox` covering ≥ 30% of viewport *or* containing an element with `role="dialog"`/`role="alertdialog"`.
2. For each candidate, classify by text + role + class signals:
   - `cookie_banner`: text matches `/cookie|consent|gdpr/i` + has an accept/dismiss button.
   - `auth_required`: text matches `/sign in|log in|session/i`.
   - `marketing`: has a "no thanks" or "later" CTA + promotional language.
   - `error`: text matches `/error|sorry|something went wrong/i`.
   - `confirmation`: text matches `/are you sure|confirm/i` + has confirm/cancel pair.
   - `unknown`: anything else.
3. Returns `{ modals: [{ ref, classification, confidence (0–1), dismissible, text_summary (≤200 chars) }] }`.

The classifier is intentionally heuristic. Jon makes the final call. Auto-dismissal is *not* in this ticket — TB-1-15's state pack opts into auto-dismiss for cookie banners with confidence > 0.9 only.

**Acceptance.** Fixture page `/aa/dashboard?cookie=1` shows the cookie banner; `detect_modal` returns one modal with classification `cookie_banner`, confidence > 0.9. `/aa/timeout` returns one modal classified `auth_required` (the session-expired pattern). `/aa/dashboard` with no modal returns `[]`.

**Dependencies.** TB-1-1, TB-1-2.

---

### TB-1-9 — `detect_error`
**Owner:** Ext Dev
**Complexity:** S

**Description.** Per design §3.4. Returns one of:
- `none`: no error signals.
- `http_4xx` / `http_5xx`: HTTP status from the navigation hook (TB-1-4 stores the last status per tab).
- `access_denied`: DOM signals `403`/`access denied`/`not authorized` + path doesn't contain `/login`.
- `login_expired`: text patterns `/session.*expired|please log in again/i` *or* a redirect-to-login URL in the last navigation.
- `unknown`: nothing matched but the page is empty/minimal.

Returns `{ kind, evidence: { url, status_code?, matched_text? } }`.

**Acceptance.** `/aa/timeout` → `login_expired`. `/aa/error-500` (fixture variant) → `http_5xx`. `/aa/dashboard` (clean) → `none`.

**Dependencies.** TB-1-1, TB-1-4.

---

### TB-1-10 — `detect_loading`
**Owner:** Ext Dev
**Complexity:** S

**Description.** Per design §3.4. CS reports `{ loading: bool, signals: [...] }`. Signals checked:
- `document.readyState !== "complete"`.
- ≥ 1 pending fetch in the last 500 ms (TB-1-4's `PerformanceObserver` wrapper).
- A spinner element matching common selectors (`[role=progressbar]`, `.spinner`, `.loading`, `.loader`) with non-zero bbox.

**Acceptance.** During a fixture XHR storm: `loading: true`, signals lists `pending_fetch`. Idle page: `loading: false, signals: []`.

**Dependencies.** TB-1-1, TB-1-4.

---

### TB-1-11 — `is_logged_in` (cookie presence only)
**Owner:** Ext Dev
**Complexity:** S

**Description.** Per design §3.5 and §5.2.

SW-side check (CS contributes the DOM half):
1. `chrome.cookies.getAll({ domain })` — returns presence + expiry only. **Cookie values are never read or logged.** A `cookieValueGuard` test in CI fails the build if any code path reads `.value`.
2. CS checks for site-specific anchors. For `aa.com`: `a[href*="aadvantage"]` with non-empty text *or* `[data-aa-user]` landmark. For the PBS domain (configurable): a pilot-name landmark `[data-pbs-pilot]` or a role=banner element containing the configured "Logged in as" text pattern.
3. Combine: `logged_in: true` iff session cookie present (name match against site config) **and** at least one DOM anchor matches. Returns `{ logged_in, evidence: "cookie" | "dom" | "url" }`. The `evidence` field reports the strongest single signal so Jon can decide whether to trust a partial.

**Acceptance.** Fixture page `/aa/dashboard` with session cookie set → `logged_in: true`. Same page with cookie deleted → `logged_in: false`. CI guard: grep for `cookie.value` outside the `is_logged_in` cookie-name reader returns zero hits.

**Dependencies.** TB-1-1, TB-1-13 (allowlist gate).

---

### TB-1-12 — Audit log: local IndexedDB chain
**Owner:** Ext Dev
**Complexity:** L

**Description.** Per design §4.4 and §7.6.

Every completed action emits an Action Record into the `audit` store (TB-0-2). Each record carries `prev_hash = sha256(canonical(prior_record))`. The chain head is held in memory and persisted on every write; chain integrity is reasserted on SW wake.

Device signature: per design §4.4, each record is signed by the device key (TB-0-7). The signature covers the canonicalized JSON of the record (sorted keys, fixed encoding) so the signature is independent of JSON formatting.

Upstream flush: a `flushAudit` step (driven by the heartbeat alarm at 25 s) batches unflushed records into an `audit` event upstream. The bridge (TB-0-6) acks with `last_accepted: <hash>`; only then does the SW mark records `server_acked: true`. Local rotation policy keeps the most recent 1,000 records or 7 days, whichever is greater, **and never drops records with `server_acked: false`** even if the cap is exceeded — those raise an `audit_backpressure` event upstream instead.

Phase 1 wires the chain, the signing, and the upstream flush. Phase 2 will add the gateway anchor (the bridge already writes anchors per TB-0-6).

**Acceptance.** A run that issues 10 `click`s produces 10 chained records. Tamper test: edit one record in IndexedDB directly via DevTools → next chain re-assertion fails and emits `audit_tampered` event upstream. Disconnect test: 5 actions while bridge is down → records persist locally → on reconnect, all 5 flush and get acked → `server_acked: true` for all. Tampered server-ack: bridge returns a wrong `last_accepted` hash → SW does not mark records acked, emits `audit_chain_mismatch`, pauses run.

**Dependencies.** TB-0-2, TB-0-5, TB-0-6, TB-0-7.

---

### TB-1-13 — Domain allowlist enforcement (defense in depth)
**Owner:** Ext Dev
**Complexity:** M

**Description.** Per design §7.4. Two-layer enforcement:

Layer 1 (manifest): `content_scripts.matches` and `host_permissions` derive from the allowlist persisted in IndexedDB at pairing. Re-derive on allowlist change (options-page edit) and reload the manifest via `chrome.runtime.reload` (with a confirm dialog because reload disconnects WSS briefly).

Layer 2 (SW code): every command's resolved `tab_id` is checked against the allowlist by origin before any CS message is sent. Commands targeting a non-allowlisted origin return `error { code: "NOT_ALLOWLISTED", retriable: false }` immediately, with the origin redacted to `<host>` (not full URL) in logs to avoid leaking path-level intent.

**ThunderGate cannot expand the allowlist remotely.** The protocol has no `set_allowlist` command and the bridge must not introduce one. Adding a host requires Michael to edit the options page. This is the architectural lever against a compromised TG (design §7.2).

**Acceptance.** Allowlist = `["aa.com", "*.aa.com", "<pbs>", "localhost:7860"]`. `navigate` to `https://example.com` → `NOT_ALLOWLISTED`. `navigate` to `https://www.aa.com/dashboard` → allowed. Try to add to the allowlist via a crafted WSS message: bridge has no handler; if a future commit accidentally adds one, the CI test `tests/allowlist-immutable.spec.ts` (added in this ticket) fails.

**Dependencies.** TB-0-2.

---

### TB-1-14 — Input redactor (CS-side, runs before any data leaves)
**Owner:** Ext Dev
**Complexity:** M

**Description.** Per design §7.3. Lives in the CS and runs on every snapshot and every `get_form_state` output before the data crosses to the SW.

Hardcoded rules (cannot be disabled via UI):
- `input[type=password]` → value replaced with `"[REDACTED:password]"`, length recorded as `value_len`.
- `input[autocomplete^="cc-"]`, `input[name*="card"]` (case-insensitive), `input[name*="cvv"]`, `input[name*="cvc"]` → `"[REDACTED:cc]"`.
- `input[autocomplete*="ssn"]` and text nodes matching `/\b\d{3}-\d{2}-\d{4}\b/` → `"[REDACTED:ssn]"`.
- Text nodes matching Luhn-valid 13–19 digit sequences → `"[REDACTED:cc]"`.

User-extensible (via options page only, never over WSS):
- A regex denylist; each pattern compiled and applied to every text node and input value.

The redactor is a pure function; it has its own unit tests (`tests/redactor.spec.ts`) with a corpus including: a password field with React-controlled value, a CC field hidden behind a custom component, an SSN written into a `<p>`, a denylist regex for a hypothetical site-specific token pattern.

Important invariant: redaction happens **before** the CS sends the message to the SW. The SW receives only redacted data. ThunderGate definitely doesn't see plaintext secrets. The integration test for this asserts on the SW side that no message body contains a string matching any of the hardcoded secret patterns.

**Acceptance.** Redactor unit tests pass. Integration: fill the password field on `/aa/login` with "hunter2", then `snapshot_dom` — the snapshot shows `[REDACTED:password]`, length 7, and `hunter2` does not appear anywhere in the SW message logs (CI grep guard).

**Dependencies.** TB-1-1, TB-1-2.

---

### TB-1-15 — State-pack format + AA fixture state detector
**Owner:** Ext Dev + CLI Jon
**Complexity:** L

**Description.** Per design §5.1. A "state pack" is a versioned JSON ruleset shipped with the extension and hot-updateable via TG push (the push is Phase 2; Phase 1 just supports a local file).

Schema:
```
{
  "pack_id": "aa-v1",
  "version": 1,
  "states": [
    {
      "id": "aa.dashboard",
      "entry_detectors": [
        { "kind": "url", "pattern": "^https://.*\\.aa\\.com/(?:dashboard|account)", "weight": 0.5 },
        { "kind": "dom", "selector": "[data-aa-user]", "weight": 0.5 }
      ],
      "min_confidence": 0.8,
      "expected_actions": ["click", "snapshot_dom", "query"],
      "exits": [
        { "to": "aa.travel_planner_empty", "trigger": "click on role=link name='Travel planner'" }
      ]
    },
    ...
  ]
}
```

Phase 1 ships the AA pack covering: `aa.unauth`, `aa.dashboard`, `aa.travel_planner_empty`, `aa.search_loading`, `aa.search_results`, `aa.fare_selection`, `aa.passenger_info`, `aa.payment`, `aa.confirm_review`, `aa.confirmed`, `aa.password_change`, `aa.timeout`, `aa.captcha_blocked`, `aa.error`. The CS evaluates entry detectors on every coalesced DOM mutation and emits `state_detected` events when confidence ≥ `min_confidence`.

Fixture pages built in parallel (CLI Jon): each AA state has a corresponding `fixtures/aa/<state>.html` per design §8.3 list. Fixtures intentionally include realistic noise: cookie banners, marketing modals, slow XHRs, the confirm button's stability quirks (two rapid re-renders).

**Acceptance.** Walking through the AA fixture states emits `state_detected` events with the right `id` in order. `aa.confirmed` is reached only after the precision-click on the `aa.confirm_review` button. Confidence scores logged for inspection; none below 0.85 on the happy path.

**Dependencies.** TB-1-1 through TB-1-11, TB-0-11.

---

### TB-1-16 — `tg-mock` AA fixture scripts
**Owner:** CLI Jon
**Complexity:** M

**Description.** Five `.tbscript` files driving the AA fixture site via the dev extension, per design §8.4:

- `aa_happy_path.tbscript` — login (manual seed of session cookie) → dashboard → travel planner → search → fare → passenger → payment → confirm_review → precision-click confirm (with simulated per-action confirm ack inline since BYOAA is Phase 2) → confirmed.
- `aa_sold_out_replan.tbscript` — search returns a "sold out" variant; Jon re-queries with different params.
- `aa_password_expired.tbscript` — mid-flow navigation lands on `/password-expired`; expect `state_detected: aa.password_change`, then `pause` event.
- `aa_session_timeout_midflight.tbscript` — `/timeout` modal appears between fare selection and passenger info; expect `error_detected: login_expired`, pause.
- `aa_captcha_dead_end.tbscript` — captcha fixture; expect `state_detected: aa.captcha_blocked`, pause, no auto-solve attempt.

Each script ends with `ASSERT no unredacted secrets in any message` (the SW message log is captured during the run and grep'd post-hoc).

CI integration: `npm run fixtures:aa` runs all five; exit 0 only if all pass.

**Acceptance.** All five scripts pass green locally and in CI. The CI Doctor check (TB-0-11 line item) covers fixture-pass = 100% on every PR.

**Dependencies.** TB-0-10, TB-1-15.

---

### TB-1-17 — `tg-mock` PBS fixture scripts + PBS state detector
**Owner:** CLI Jon + Ext Dev
**Complexity:** L

**Description.** Parallel to TB-1-15/16 but for the PBS portal. The PBS state machine is specified in §3 of this document. Fixture pages cover every PBS state. Three scripts at minimum:

- `pbs_happy_path.tbscript` — login → dashboard → open bid window → load grid → set preferences (rank 1 through N) → review ballot → precision-click save → ballot saved.
- `pbs_window_closed.tbscript` — Jon opens the bid window after close; portal returns "bidding closed" view; expect `state_detected: pbs.window_closed`, pause, narration to Michael.
- `pbs_preferences_invalid.tbscript` — fixture rejects the ballot with "duplicate rank" error; expect `state_detected: pbs.preferences_invalid`, pause for Michael to resolve.

State pack `pbs-v1` ships alongside `aa-v1`. The state IDs match §3 exactly.

**Acceptance.** All three PBS scripts pass green. The state pack detects each PBS state at min confidence 0.85 on the happy path. The save-ballot precision click is gated by the same stability check as the AA confirm click.

**Dependencies.** TB-1-15, TB-1-16, §3 of this document (PBS FSM).

---

### TB-1-18 — Recording mode (`.tbrec` dump + replay tool)
**Owner:** Ext Dev
**Complexity:** M

**Description.** Per design §8.5. When a run is started with `{ record: true }`, the SW dumps every action and the corresponding DOM-before/after (full snapshots, not just hashes) to a `.tbrec` JSONL file via `chrome.downloads`. A separate `scripts/tbrec-replay.ts` tool loads a `.tbrec` and replays it against the fixture site, asserting that the same diffs come out.

This is the Ghost-Jon-equivalent regression harness, as called out in the design. It seeds the eventual shadow-eval for the extension (open question §10.5 of the design — flagged for follow-up, not in scope for Phase 1).

**Acceptance.** Record the `aa_happy_path` run → produces a `.tbrec`. Replay against the fixture → exit 0 with all assertions passing. Replay against a fixture variant (e.g., changed button label on confirm) → fails with a diff report pointing to the changed element.

**Dependencies.** TB-1-2, TB-1-12, TB-1-15, TB-1-16.

---

### Phase 1 exit gate

1. TB-1-1 through TB-1-18 marked done.
2. All AA fixture scripts (TB-1-16) and PBS fixture scripts (TB-1-17) green in CI.
3. Recording mode (TB-1-18) produces a replayable `.tbrec` for the happy paths of both AA and PBS.
4. Audit chain (TB-1-12) integrity test passes locally and via the bridge anchor.
5. Redactor CI guard (TB-1-14) green.
6. Allowlist immutability CI guard (TB-1-13) green.
7. Doctor checks extended (design §8.7): WSS reconnect rate, audit chain unbroken, fixture pass rate, no unexpected `SCOPE_DENIED` (will be vacuously true until Phase 2), no `cs_disconnected` > 5 s.
8. End-to-end demo on the fixture site of both flows, narrated by Michael as if it were the live portal — confirming the ergonomics work *before* any real-portal connect.

Only when all eight pass does Phase 2 (BYOAA) start.

---

## 3. PBS Automation State Machine (parallel to AA FSM in design §5)

This is the design surface Phase 1's PBS fixtures (TB-1-17) implement. Phase 1 does not connect to the real PBS; it only proves the state machine and the action surface against fixtures. Real-PBS dry-run is Phase 3-equivalent for PBS and will be filed as TB-3-PBS when AA Phase 3 lands.

### 3.1 What PBS is

PBS = the airline's Pilot Bidding System. Pilots submit a ranked ballot of pairings (multi-day trips) for an upcoming bid period. The system is stateful, has a hard window (open/close timestamps), and the ballot is binding once submitted until the next window opens. Jon's job: read the open bid slots, fill Michael's preferences in the order Michael specifies, walk the workflow, and stop at the precision-click "Save Ballot" gate for Michael's ThunderCommo confirmation. Same live DOM + real-time reaction model as AA — but PBS has its own state shape, its own failure modes, and a higher-cost mistake (a wrong ballot can't be unsaved mid-window without manual recourse).

### 3.2 State diagram

```
[pbs.unauth]
    │  login event (user-driven; extension does not fill credentials in v1)
    ▼
[pbs.dashboard]
    │  click "Bid for next period"  (role=link name=/bid|ballot/i)
    ▼
[pbs.window_check]                   ← entry detector verifies dates+clock
    │  window open + clock within bounds        window closed / not yet open
    ├──────────────────────────────▶  [pbs.bid_window_open]
    └──────────────────────────────▶  [pbs.window_closed]   (pause, narrate)
                                       │
                                       ▼
                                     (no exits — Michael takes over)

[pbs.bid_window_open]
    │  click "Load pairings" or wait for autoload
    ▼
[pbs.bid_grid_loaded]                ← grid of pairing rows visible
    │  set preferences (one fill per row, optionally one click per rank dropdown)
    ▼
[pbs.preferences_set]
    │  click "Review ballot"
    ▼
[pbs.ballot_review]
    │  validation pass                          validation fail
    ├──────────────────────────────▶ [pbs.ready_to_save]
    └──────────────────────────────▶ [pbs.preferences_invalid]
                                       │
                                       └─ fix-up loop back to bid_grid_loaded

[pbs.ready_to_save]
    │  precision-click "Save Ballot"  ← requires ThunderCommo confirmation in Phase 2
    ▼
[pbs.ballot_saved]                    ← confirm: success banner + ballot ID in DOM

Anywhere ── session timeout ──▶ [pbs.unauth]
Anywhere ── soft warning ─────▶ [pbs.soft_warning]   (e.g., "your last pairing
                                  conflicts with vacation request") — surface to
                                  Jon, do not auto-dismiss
Anywhere ── error ────────────▶ [pbs.error]
```

### 3.3 State-by-state spec

| State | Entry detector | Expected actions | Exit triggers |
|---|---|---|---|
| `pbs.unauth` | URL pattern `/login` OR a "Sign in" form is the dominant landmark | `is_logged_in`, `snapshot_dom`, nothing else | login event detected (cookie + dashboard anchor) |
| `pbs.dashboard` | URL pattern `/dashboard\|/home` + `[data-pbs-pilot]` landmark OR text "Welcome, <name>" | `click` on bid nav, `snapshot_dom`, `query`, `get_text` | nav click |
| `pbs.window_check` | URL pattern `/bid|/ballot` + presence of a window-open timestamp element | `get_text` to read window open/close + clock | window state determination |
| `pbs.window_closed` | text matches `/bidding (is )?closed|window not (yet )?open/i` | `snapshot_dom`, pause, narrate | none (Michael takes over) |
| `pbs.bid_window_open` | text matches `/bid period open|submit your bids/i` + "Load pairings" button present | `click` load, `snapshot_dom` | grid load detected |
| `pbs.bid_grid_loaded` | A grid element `[data-pbs-grid]` OR ≥ 10 rows with class containing `pairing-row` | `query` for rows, `fill` for preference rank, `select`/`check` per the portal's input style, `scroll_to` | "Review ballot" clicked |
| `pbs.preferences_set` | DOM signal that ranks have been applied — typically a numeric badge per row | `click` review, `snapshot_dom` | review click |
| `pbs.ballot_review` | URL pattern `/review\|/preview` + a summary block | `snapshot_dom`, `query`, `get_text` | validation outcome detected via state |
| `pbs.preferences_invalid` | Error banner text matches `/invalid|duplicate|conflict/i` OR validation row marked red | `query` for the offending row, narrate, pause | Michael resolves; loop back |
| `pbs.ready_to_save` | "Save Ballot" button present + enabled + a clean preview | `precision-click` save (Phase 2: requires `require_user_confirmation_for`) | save click |
| `pbs.ballot_saved` | Success banner OR ballot ID landmark `[data-pbs-ballot-id]` | `get_text` to capture the ballot ID for audit | terminal |
| `pbs.soft_warning` | Yellow banner OR text "warning" not in an error pattern | `snapshot_dom`, narrate, do not auto-dismiss | Michael acks |
| `pbs.error` | Red banner OR HTTP 5xx + DOM evidence | `snapshot_dom`, narrate, pause | Michael resolves |

### 3.4 The PBS confirmation gate

Same shape as AA's `confirm_review` precision click, with three differences:

1. **Higher action budget required.** A PBS ballot fill can be 50-200 actions for a long pairing month. Scope tokens for PBS default to `max_actions: 400`.
2. **Stricter stability check.** The Save Ballot button gets a 500 ms (not 250 ms) double-query stability check because PBS portals are notorious for late-rendering disabled/enabled toggles based on async validation.
3. **Post-save evidence requirement.** After save, the extension must capture the ballot ID from the DOM and store it in the audit record's `evidence` field. If no ballot ID appears within 15 s, emit `error_detected` and pause — the ballot may or may not have saved, and Michael needs to verify manually.

### 3.5 What PBS automation does NOT do in v1

- **No credential entry.** Same posture as AA.
- **No automatic re-bid after window close.** If the window has closed, Jon does not attempt to re-open or escalate.
- **No bulk pairing import from a spreadsheet.** Future ticket; v1 fills row-by-row from a JSON preference list sent in the start_run params.
- **No automatic conflict resolution.** Soft warnings and validation failures pause for Michael.

### 3.6 Why PBS lands in Phase 1 fixtures alongside AA

Because the use case is structurally identical — live DOM walking, precision-click save, narration to Michael — and getting both fixtures green forces the state-pack format (TB-1-15) to actually be generic instead of accidentally hardcoded to AA shapes. If we only built AA in Phase 1 and PBS in Phase 4, we'd waste a week on the second integration discovering the first hardcoded too many assumptions. Cheaper to surface that now.

---

## 4. ThunderCommo Live Narration Spec

Michael watches his own Chrome screen in real time. Jon drives the browser. ThunderCommo is the audio/text channel Jon uses to tell Michael what's happening, what's next, and what he wants Michael's eyes on. This spec defines what Jon says, when, and what events trigger it.

### 4.1 Narration posture

Three principles, in order:

1. **Confirm-before-irrevocable.** Every precision-click (AA confirm, PBS save ballot, anything marked `require_user_confirmation_for`) requires Michael's tap. The narration before the tap is not a status update — it's the actual ask.
2. **Tell Michael where his eyes should be.** Michael is watching his own browser. Jon's job is to point. "Look at the upper-right corner where the modal just appeared" beats "modal detected."
3. **Stay quiet during routine steps.** Narrating every click is noise. Narrate state transitions, not actions inside a state. The default cadence is one narration per state change plus one per warning/error.

### 4.2 Channel mechanics

Two streams:

- **Speech** (TTS to Michael's phone): short, declarative, ≤ 12 words per utterance. Used for state transitions and confirmation asks. Tagged with priority `routine|warning|confirmation|critical` for the iOS app's audio profile.
- **Text** (push notification + in-app banner): used for non-time-critical events (audit anchor written, scope warning at T-60s, recording stopped). Also used as the long-form companion to a speech utterance ("PBS preferences set" speech + the row-by-row preference list as text).

Confirmation asks always go on speech, never on text alone. A buzz-and-tap pattern: speech says "Ready to save ballot. Tap confirm when ready." — text shows the ballot ID and the precise button being clicked. Tap is captured by ThunderCommo and round-tripped to TG as a fresh per-action confirmation token (design §4.3).

### 4.3 Event-to-narration table

| Event (extension → bridge) | Speech to Michael | Text companion | Priority |
|---|---|---|---|
| `run started` (start_run) | "Starting <intent>. Estimated <N> steps." | Run ID + scope intent + estimated duration | routine |
| `state_detected: aa.dashboard` | "On the AA dashboard." | URL + state pack id + confidence | routine |
| `state_detected: aa.travel_planner_empty` | "Travel planner is open. Ready to search." | — | routine |
| `state_detected: aa.search_results` | "Got <N> results. Reviewing." | Result count + first three options | routine |
| `state_detected: aa.fare_selection` | "Selecting fare." | Fare class + price | routine |
| `state_detected: aa.passenger_info` | "Filling passenger info." | Field names being filled (no values) | routine |
| `state_detected: aa.payment` | (silent — Jon does not fill payment) | "Payment screen reached — your hands from here, or extend scope." | warning |
| `state_detected: aa.confirm_review` + `require_user_confirmation_for` triggered | "Ready to confirm <trip>. Tap to confirm." | Full trip summary + price + the exact button | confirmation |
| `state_detected: aa.confirmed` | "Confirmed. Confirmation code <code>." | Full confirmation block | critical |
| `state_detected: pbs.dashboard` | "On the PBS dashboard." | — | routine |
| `state_detected: pbs.window_closed` | "Bid window is closed. Stopping." | Close timestamp + next window | warning |
| `state_detected: pbs.bid_grid_loaded` | "Pairings loaded. Setting your preferences." | Number of pairings + Michael's preference list | routine |
| `state_detected: pbs.preferences_set` | "Preferences set. Reviewing ballot." | — | routine |
| `state_detected: pbs.preferences_invalid` | "Ballot has a conflict on row <N>. Look at row <N>." | The validation message verbatim | warning |
| `state_detected: pbs.ready_to_save` | "Ready to save ballot. Tap confirm to commit." | Full ballot preview (top-10 rows + total) + the exact button | confirmation |
| `state_detected: pbs.ballot_saved` | "Ballot saved. ID <ballot_id>." | Full ballot record + audit chain head | critical |
| `modal_appeared: cookie_banner` | (silent if classifier confidence > 0.9 and auto-dismiss enabled) | — | routine |
| `modal_appeared: auth_required` | "Session expired. Please log back in." | Login URL | warning |
| `modal_appeared: confirmation` | "Site is asking you to confirm. Eyes on the dialog." | Dialog text | warning |
| `modal_appeared: marketing` | (silent; Jon ignores) | — | routine |
| `modal_appeared: unknown` | "Unexpected modal. I'm not touching it." | Modal text summary | warning |
| `error_detected: login_expired` | "Logged out. I need you to log back in." | Last URL | warning |
| `error_detected: http_5xx` | "Site returned an error. Pausing." | Status + URL | warning |
| `error_detected: access_denied` | "Site denied access. Pausing." | URL + evidence | warning |
| `state_detected: aa.password_change` | "AA password expired. You take it from here." | URL | warning |
| `state_detected: aa.captcha_blocked` | "CAPTCHA. Stopping — your turn." | URL | warning |
| `scope_warning: expires_soon` (T-60s) | "Scope expires in one minute. Extend?" | Scope ID + remaining actions | warning |
| `scope_warning: action_count_near_limit` | "Almost out of actions on this scope. Extend?" | Used / limit | warning |
| `scope_expired` | "Scope expired. Run is parked." | Scope ID + run ID | warning |
| `scope_revoked` | "Scope revoked. Stopping." | Scope ID | critical |
| `audit anchor written` (60 s tick) | (silent) | (silent — visible in audit viewer only) | routine |
| `cs_disconnected` > 5 s | "Lost the page. Reconnecting." | Tab + URL | warning |
| `run paused` (manual or auto) | "Paused." | Reason | routine |
| `run aborted` | "Aborted: <reason>." | Full reason + audit chain head | critical |

### 4.4 Confirmation envelope

When a precision-click is reached, the bridge emits a confirmation request to ThunderCommo:

```
{
  type: "event",
  body: {
    kind: "confirmation_required",
    scope_id, run_id, action_id,
    action: "click",
    summary: {
      title: "Save PBS ballot for May 2026 bid period",
      preview: "<text summary of what is about to happen>",
      irrevocable: true,
      target_url, target_element_label
    },
    expires_in_sec: 30
  }
}
```

ThunderCommo renders a confirm/cancel pair. Michael's tap signs a per-action confirmation token with the phone key (per design §4.3). The token is round-tripped to TG, which forwards it to the bridge, which authorizes the dispatch. If 30 s elapses with no tap, the bridge emits `confirmation_timeout`, narration says "Confirmation timed out. Pausing," and the run pauses.

### 4.5 Narration suppression rules

To keep the channel useful and not chatty:

- Repeat suppression: if the same speech utterance fires within 10 s, the second one is dropped (text companion still goes through).
- During a high-frequency state (e.g., `pbs.bid_grid_loaded` while Jon fills 100 rows), narration fires once at state entry and once at exit, not per fill.
- During a `pause` state, no routine narration. Warnings and confirmations still go through.
- Michael has a global "quiet mode" toggle in ThunderCommo that drops everything below `critical` priority. Even in quiet mode, confirmations override.

### 4.6 What Jon does NOT narrate

- Individual field fills (too noisy).
- Audit chain anchor events (background bookkeeping).
- DOM mutations smaller than a state change.
- Successful heartbeats / reconnects under 5 s.
- Cookie banners auto-dismissed with high confidence.
- Page-internal navigation that doesn't change state.

The bias is heavy: when in doubt, don't narrate. Michael is watching the screen. Jon's voice should mean something every time it fires.

### 4.7 Tone

CLI Jon's prose voice carries to the speech track. Direct, declarative, no filler. "Ready to save ballot. Tap confirm." not "I'm about to attempt to save the ballot. If you would like to proceed, please tap the confirm button." Twelve words is the cap because anything longer interrupts what Michael is doing.

The voice profile is the same as ThunderCommo's existing voice (per `THUNDERCOMMO_BUILD24_GATE_BRIEF.md` and related). No new voice or persona — Jon-the-browser is Jon-the-rest-of-Jon.

---

## 5. Open items (carry to the next stand-up)

1. **PBS domain.** The allowlist seed in TB-0-8 includes `<airline_pbs_domain>`. The actual value is config-driven on the TG side and must be confirmed with Michael before Phase 0 demo. Flagged here, not in a separate doc, because the value is short-lived secret-ish.
2. **PBS preference input format.** The state pack assumes `fill` per row for rank values. If the real PBS portal uses drag-and-drop reordering instead of numeric fills, Phase 1's PBS scripts will need a `drag_to` action — which is *not* in Phase 1's surface. We resolve by walking the real portal in shadow mode (design §8.6 mode 2) early in Phase 3-PBS planning, not by guessing now.
3. **ThunderCommo confirmation token signing.** Phase 2's BYOAA work depends on the ThunderCommo iOS side already having phone-key signing (`THUNDERCOMMO_IOS_BUILD24_BRIEF.md`). Confirm with Mack that the build that lands phone-key signing is on the Phase 2 path, not delayed past it. If delayed, the precision-click gate in Phase 1 fixtures will be stubbed (a synchronous "tap confirm" event in the mock) and the BYOAA Phase 2 gate becomes the real bar.
4. **State pack hot-update.** Design §5.1 hints at TG pushing state pack updates over WSS. Phase 1 only loads local packs. The push protocol is a Phase 2+ ticket; spec the message shape now or later? Recommend later — design the push the same week the first AA DOM change forces a state-pack revision (i.e., on real-portal pain, not speculatively).
5. **Patent prep.** Per design §10.4, keep the KYA-scoped action-record chain (TB-1-12 + TB-0-6 anchor) private until the provisional filing. Tickets are fine internally; do not link tickets in public PRs.

---

*CLI Jon | ThunderBase | 2026-05-11*
