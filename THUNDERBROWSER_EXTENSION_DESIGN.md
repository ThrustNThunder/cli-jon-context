# ThunderBrowser Extension — Design Document
**Author:** CLI Jon (ThunderBase)
**Date:** 2026-05-11
**Status:** Design only. No code. This becomes the build brief for the extension developer (or for Mack/CLI Jon when we cut the first ticket).
**Reference:** `project_jon/THUNDERBROWSER.md`, `THUNDERBROWSER_EXTENSION_BRIEF.md`

---

## 0. Design Posture

ThunderBrowser is not a scraper, not a headless automation rig, not an LLM clicking screenshots. It is a Chrome Manifest V3 extension running inside Michael's real browser with Michael's real session cookies, talking to ThunderGate over an authenticated WebSocket. The browser is the trust boundary — the extension borrows Michael's identity by living in his profile, and BYOAA records that the agent acted under a specific, time-limited, scope-limited authorization.

The design priorities in order:

1. **Don't break Michael's browser.** Fail closed. Never silently sniff cookies or passwords. Never act outside the authorization scope.
2. **Be observable.** Every action emits a DOM-state-before/after record that ThunderGate persists. Court-admissibility is the ceiling — Doctor-style audit is the floor.
3. **Be live, not screenshot-y.** The extension streams structured DOM state; Jon reasons over structured data, not OCR.
4. **Be replaceable.** Safari Web Extension is a wrapper, not a fork. APIs go through a thin platform shim so 90%+ of the codebase is shared.
5. **Be small first.** MVP is pairing → connect → navigate/click/fill/snapshot → AA fixture. Everything else is Phase 2+.

---

## 1. Extension Component Map

```
┌───────────────────────────────────── Michael's Mac / Chrome ─────────────────────────────────────┐
│                                                                                                  │
│   ┌────────────────────┐         ┌──────────────────────────┐                                    │
│   │  Popup UI          │         │  Options Page            │                                    │
│   │  - status pill     │         │  - pairing flow (QR)     │                                    │
│   │  - current scope   │         │  - domain allowlist      │                                    │
│   │  - pause/abort     │         │  - audit log viewer      │                                    │
│   │  - revoke          │         │  - device key info       │                                    │
│   └─────────┬──────────┘         └────────────┬─────────────┘                                    │
│             │ chrome.runtime.sendMessage      │                                                  │
│             └──────────────┬──────────────────┘                                                  │
│                            │                                                                     │
│                  ┌─────────▼──────────┐                                                          │
│                  │ Background Service │                                                          │
│                  │ Worker (MV3)       │       ── persistent WSS ──>                              │
│                  │ - WSS client       │  ────────────────────────────────────────┐               │
│                  │ - scope state      │                                          │               │
│                  │ - action dispatcher│                                          │               │
│                  │ - audit log queue  │                                          │               │
│                  │ - alarm-driven KA  │                                          │               │
│                  └────────┬───────────┘                                          │               │
│                           │ chrome.tabs.sendMessage                              │               │
│                           │ chrome.scripting.executeScript (isolated world)      │               │
│                           │                                                      │               │
│                  ┌────────▼───────────┐                                          │               │
│                  │ Content Script     │                                          │               │
│                  │ (per allowed tab)  │                                          │               │
│                  │ - DOM read/write   │                                          │               │
│                  │ - MutationObserver │                                          │               │
│                  │ - modal detector   │                                          │               │
│                  │ - input redactor   │                                          │               │
│                  └────────┬───────────┘                                          │               │
│                           │                                                      │               │
│                  ┌────────▼───────────┐                                          │               │
│                  │ Page DOM           │                                          │               │
│                  └────────────────────┘                                          │               │
│                                                                                  │               │
│   Local IndexedDB:                                                               │               │
│     - audit log (chained hash)                                                   │               │
│     - device keypair (non-extractable, crypto.subtle)                            │               │
│     - last good scope token (encrypted)                                          │               │
│                                                                                  │               │
└──────────────────────────────────────────────────────────────────────────────────┼───────────────┘
                                                                                   │
                                              wss://gateway.thrustnthunder/browser │
                                                                                   │
┌──────────────────────────────────────── ThunderBase / EC2 ───────────────────────▼───────────────┐
│                                                                                                  │
│   ┌─────────────────────────────┐                                                                │
│   │  Browser Bridge (new)       │   src/browser/bridge.ts                                        │
│   │  - WSS server               │                                                                │
│   │  - per-extension session    │                                                                │
│   │  - command queue + reply    │                                                                │
│   │  - audit ingest             │                                                                │
│   └────────┬───────────┬────────┘                                                                │
│            │           │                                                                         │
│   ┌────────▼─┐   ┌─────▼──────┐    ┌───────────────┐    ┌──────────────────┐                    │
│   │context.db│   │BYOAA       │    │ Jon reasoning │    │ ThunderCommo     │                    │
│   │(actions, │   │authority   │←──▶│ loop          │    │ (Michael's phone │                    │
│   │ scopes,  │   │service     │    │ (LLM)         │    │  for scope grant │                    │
│   │ audit)   │   └────────────┘    └───────────────┘    │  + revoke)       │                    │
│   └──────────┘                                          └──────────────────┘                    │
│                                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Components

| Component | Lifetime | Responsibility | Why it lives here |
|---|---|---|---|
| **Background SW** | MV3 ephemeral (woken by event/alarm) | WSS connect, scope state, dispatch | Only place with cross-tab visibility and persistent state |
| **Content script** | Per-tab, per-load | DOM read/write, mutation observation | Only place that can touch the DOM |
| **Popup** | Open while user clicks toolbar icon | Status, manual pause/abort | UX surface |
| **Options page** | On-demand | Pairing, allowlist, audit viewer | Setup + transparency |
| **IndexedDB** | Persistent | Audit log, keypair handle, pairing | SW restarts wipe in-memory state; need durability |
| **Browser Bridge (server)** | Long-lived | WSS endpoint, command queue, audit ingest | New TG module — keeps existing channels untouched |

### 1.2 MV3 service-worker reality

MV3 service workers terminate after ~30s idle. Design accommodations:

- **`chrome.alarms`** every 25s wakes SW for WSS heartbeat (do NOT rely on a `setInterval` — it dies with the worker).
- **WSS reconnect on wake**: every entry into the SW checks WSS state; reconnect if closed. ThunderGate side keeps last 30s of commands in a per-extension queue (drained on reconnect).
- **State persistence**: any state that must survive an SW restart goes to `chrome.storage.session` (memory-backed, survives some restarts) or IndexedDB (durable). Scope token, current run ID, last action ID — all persisted.
- **No DOM in SW**: SW cannot access DOM, `window`, or `document`. All DOM work goes through content scripts via `chrome.tabs.sendMessage` or `chrome.scripting.executeScript({ world: "ISOLATED" })`.

### 1.3 Content script injection model

- **Declarative injection** in manifest for the *allowlist* domains (aa.com, etc.) so the script is present on page load and can observe redirects/interstitials immediately.
- **Programmatic injection** via `chrome.scripting.executeScript` for ad-hoc actions on tabs that aren't pre-allowlisted but have been explicitly authorized in-scope (rare; default is allowlist-only).
- Always **isolated world** — never `MAIN`. The extension does not need to share JS scope with the page and should not be observable from page-side scripts.

---

## 2. Message Protocol

### 2.1 Envelope (shared across both layers)

All messages are JSON. Single envelope shape:

```
{
  "v":     1,                          // protocol version, integer
  "id":    "<uuid v4>",                // unique per message
  "ts":    1747000000000,              // ms since epoch, sender clock
  "type":  "command" | "result" | "event" | "ack" | "error",
  "scope": "<scope_id or null>",       // BYOAA scope reference if applicable
  "ref":   "<id of message this responds to, or null>",
  "body":  { ... }                     // type-specific payload (see below)
}
```

Rules:
- Every `command` MUST receive an `ack` within 2s and either a `result` or `error` within 30s (or a `progress` event for long-running ones).
- `event` is unsolicited (DOM mutation, modal detected, scope expiry warning). No reply expected.
- `error` carries `{ code: string, message: string, retriable: boolean }`.

### 2.2 Layer A — ThunderGate ↔ Background SW (WebSocket)

**Endpoint:** `wss://gateway.thrustnthunder/browser`
**Subprotocol:** `thunderbrowser.v1`

**Connect handshake:**

1. Client opens WSS with `Sec-WebSocket-Protocol: thunderbrowser.v1` and header `Authorization: Bearer <short-lived JWT>` (the SW signs a JWT using the device key — see §7.1).
2. Server validates JWT against the paired extension's pinned public key. Rejects on mismatch.
3. Server sends `hello`:
   ```
   { type: "event", body: { kind: "hello", extension_id, server_ts, queued_commands: <n> } }
   ```
4. Client replies with `ready`:
   ```
   { type: "event", body: { kind: "ready", ua, chrome_version, allowlist, active_tabs: [...] } }
   ```
5. Server flushes any queued commands.

**Heartbeat:** Client sends `ping` event every 25s (driven by `chrome.alarms`). Server replies `pong`. Server disconnects clients silent for >60s.

**Command flow (ThunderGate → Extension):**
```
TG  -> SW : { type: "command", id: "c1", scope: "s_abc", body: { action: "click", ... } }
SW  -> TG : { type: "ack",     ref: "c1" }
SW  -> CS : (chrome.tabs.sendMessage)
CS  -> SW : (dom result)
SW  -> TG : { type: "result",  ref: "c1", body: { ok: true, dom_diff: {...} } }
```

**Audit upstream:**
```
SW  -> TG : { type: "event", body: { kind: "audit", entries: [...], chain_head: "<hash>" } }
```
Server appends to `context.db` audit table, returns `ack` with `last_accepted: <hash>`. Client doesn't drop local audit until ack.

**Disconnect/resume:** SW persists `last_command_id_acked` to IndexedDB. On reconnect, sends `resume` event with that ID. Server replays any commands with `id > last_acked`.

### 2.3 Layer B — Background SW ↔ Content Script (chrome.runtime)

Same envelope. Transport: `chrome.tabs.sendMessage(tabId, msg, callback)` for commands, `chrome.runtime.sendMessage` for content→SW upstream.

The SW maintains a map `tabId → activeRunId → contentScriptReady?`. Before dispatching, it pings the CS; if no reply in 500ms it re-injects.

**Long-running actions:** CS emits `progress` events while waiting for `wait_for_load`, `wait_for_element`, etc. SW forwards them upstream so Jon can reason about latency.

### 2.4 Versioning

`v: 1` is the only version at launch. Future versions are additive. ThunderGate negotiates the highest version both sides support via the `hello`/`ready` exchange. An extension speaking only `v: 1` must reject server messages where any required field is unknown.

---

## 3. Action API

All actions are commands sent by ThunderGate. Parameters and return types listed. Return types describe `result.body` on success; on failure the `error` envelope is returned with the codes listed.

### 3.1 Navigation

| action | params | returns | errors |
|---|---|---|---|
| `navigate` | `{ url, tab_id?, new_tab?: bool, wait_for: "load"\|"domcontentloaded"\|"network_idle", timeout_ms?: number }` | `{ tab_id, final_url, status_code }` | `NOT_ALLOWLISTED`, `TIMEOUT`, `NET_ERROR` |
| `back` | `{ tab_id }` | `{ final_url }` | `NO_HISTORY` |
| `forward` | `{ tab_id }` | `{ final_url }` | `NO_HISTORY` |
| `reload` | `{ tab_id, hard?: bool }` | `{ final_url }` | — |
| `close_tab` | `{ tab_id }` | `{ ok: true }` | — |
| `wait_for_load` | `{ tab_id, condition, timeout_ms }` | `{ load_ms }` | `TIMEOUT` |

### 3.2 DOM read

| action | params | returns |
|---|---|---|
| `snapshot_dom` | `{ tab_id, mode: "structured"\|"accessibility"\|"text", redact: true }` | `{ tree: {...}, url, title, scroll_pos }` |
| `query` | `{ tab_id, selector? , text?, role?, name?, limit?: number }` | `{ matches: [{ ref, tag, role, text, bbox, attrs }] }` |
| `get_text` | `{ tab_id, ref }` | `{ text }` |
| `get_form_state` | `{ tab_id, form_ref? }` | `{ fields: [{ ref, name, type, value_redacted, value_hash, required }] }` |
| `get_url` | `{ tab_id }` | `{ url, title }` |

Snapshot modes:
- **`structured`** — DOM tree pruned to interactive + landmark elements, attributes whitelisted, text nodes truncated to 2KB, inputs redacted (see §7.3). Default. ~10–80KB per page.
- **`accessibility`** — `document.body.ariaSnapshot()` style; semantic only. Best for LLM reasoning over forms.
- **`text`** — Plain text only. Cheapest. Use for "is the word 'Confirmed' on the page?".

`ref` is an opaque stable handle (e.g. `el#12`) the extension keeps in a per-tab map. References expire on navigation.

### 3.3 DOM write

| action | params | returns | errors |
|---|---|---|---|
| `click` | `{ tab_id, ref, modifier?: "ctrl"\|"shift", expect_navigation?: bool }` | `{ navigated: bool, final_url? }` | `ELEMENT_NOT_FOUND`, `NOT_CLICKABLE`, `OUT_OF_VIEW` |
| `fill` | `{ tab_id, ref, value, secret?: bool }` | `{ ok: true, value_hash }` | `NOT_FILLABLE`, `READONLY` |
| `select` | `{ tab_id, ref, value\|label\|index }` | `{ selected_value }` | `OPTION_NOT_FOUND` |
| `check` | `{ tab_id, ref, checked: bool }` | `{ ok }` | — |
| `scroll_to` | `{ tab_id, ref?, x?, y? }` | `{ scroll_pos }` | — |
| `press_key` | `{ tab_id, key, modifiers? }` | `{ ok }` | — |
| `hover` | `{ tab_id, ref }` | `{ ok }` | — |
| `submit_form` | `{ tab_id, form_ref }` | `{ navigated, final_url? }` | — |

`fill` with `secret: true`:
- Value is decrypted only inside the content script JIT; never logged, never in audit DOM diff, replaced by `[REDACTED]` in any snapshot.
- ThunderGate side: secret values flow through a separate codepath, never into context.db plaintext.

### 3.4 Page state

| action | params | returns |
|---|---|---|
| `detect_modal` | `{ tab_id }` | `{ modals: [{ ref, classification, dismissible, text_summary }] }` |
| `detect_error` | `{ tab_id }` | `{ kind: "none"\|"http_4xx"\|"http_5xx"\|"access_denied"\|"login_expired"\|"unknown", evidence: {...} }` |
| `detect_loading` | `{ tab_id }` | `{ loading: bool, signals: [...] }` |
| `is_logged_in` | `{ tab_id, domain }` | `{ logged_in: bool\|"unknown", evidence: "cookie"\|"dom"\|"url" }` |

Modal classifier categories: `cookie_banner`, `auth_required`, `marketing`, `error`, `confirmation`, `unknown`. The classifier is heuristic (z-index + role + text patterns) — Jon makes the final call.

### 3.5 Auth / cookies (heavily restricted)

| action | params | returns | notes |
|---|---|---|---|
| `cookie_state` | `{ domain }` | `{ has_session: bool, expires_ts?: number, names_present: [string] }` | Names only, never values. Allowlisted domains only. |
| `wait_for_session` | `{ domain, timeout_ms }` | `{ established: bool }` | Used after redirect-to-login chains |

The extension **never returns cookie values** over the wire. Period. ThunderGate sees only "is there a session, when does it expire" — never the token itself.

### 3.6 Network interception (Phase 2)

| action | params | returns |
|---|---|---|
| `install_intercept` | `{ tab_id, url_pattern, methods, expect_response_kind: "json"\|"text" }` | `{ intercept_id }` |
| `read_intercepted` | `{ intercept_id, limit }` | `{ entries: [{ url, status, body_redacted }] }` |
| `remove_intercept` | `{ intercept_id }` | `{ ok }` |

Patterns must match an allowlisted domain. Bodies pass through the same redactor as form values.

### 3.7 Lifecycle

| action | params | returns | notes |
|---|---|---|---|
| `start_run` | `{ scope_id, label, expected_tabs?, expected_actions?: number }` | `{ run_id }` | Opens a run grouping for audit |
| `pause` | `{ run_id, reason }` | `{ paused_at }` | SW stops dispatching, Popup turns yellow |
| `resume` | `{ run_id }` | `{ ok }` | |
| `abort` | `{ run_id, reason }` | `{ ok }` | Discards in-flight; final audit flush; SW closes run |

### 3.8 Events (unsolicited)

| event kind | payload |
|---|---|
| `tab_navigated` | `{ tab_id, from, to, run_id? }` |
| `dom_mutation` | `{ tab_id, summary, signature }` (rate-limited; coalesced 250ms) |
| `modal_appeared` | `{ tab_id, classification, ref, text_summary }` |
| `error_detected` | `{ tab_id, kind, evidence }` |
| `scope_warning` | `{ scope_id, kind: "expires_soon"\|"action_count_near_limit", remaining }` |
| `scope_expired` | `{ scope_id }` (extension auto-pauses) |
| `cs_disconnected` | `{ tab_id }` (content script lost) |
| `audit` | `{ entries, chain_head }` |

---

## 4. BYOAA Authorization Flow

### 4.1 The token

ThunderGate issues a **Scope Token** when Michael grants authority (via ThunderCommo). Structure (signed JWT, EdDSA Ed25519, signed by ThunderGate's scope-signing key):

```
header:  { alg: "EdDSA", typ: "TBSCOPE", kid: "<key id>" }
payload: {
  iss: "thundergate://<gateway-id>",
  sub: "agent://jon",
  aud: "thunderbrowser://<extension-pair-id>",
  scope_id: "<uuid>",
  exp: <unix>,
  nbf: <unix>,
  permissions: {
    domains: ["aa.com", "*.aa.com"],
    actions: ["navigate","click","fill","select","check","scroll_to","press_key","snapshot_dom","query","detect_*","is_logged_in"],
    max_actions: 200,
    require_user_confirmation_for: ["submit_form on aa.com/booking/confirm"],
    disallow: ["cookie_state on non-allowlisted","fill secret on non-allowlisted"]
  },
  intent: "AA jumpseat reservation MIA->DFW 2026-05-13",
  intent_hash: "<sha256 of the user-facing intent string>",
  jti: "<unique>"
}
```

The extension pins ThunderGate's `kid → public key` map in IndexedDB at pairing time. Rotation is supported by signing the new key with the old one and pushing as an event before the old key is retired.

### 4.2 Grant flow

```
Michael phone (ThunderCommo)                ThunderGate                Browser (Extension)
─────────────────────────────              ──────────────              ────────────────────
1. Jon: "I need scope to handle
   AA jumpseat MIA->DFW 5/13.
   ~10 min, 50 actions max."  ────────────▶
2. Michael taps "Grant"        ────────────▶
                                            3. Mint scope token (4.1)
                                            4. Persist scope row
                                            5. Send via WSS  ─────────▶ 6. Verify sig vs pinned key
                                                                        7. Validate exp/nbf/aud
                                                                        8. Persist to chrome.storage
                                                                        9. Pop badge yellow w/ intent
10. Jon: "Acknowledged"        ◀───────────────────────────────────────
                                                                       11. Start run, dispatch actions
```

### 4.3 Per-action attestation

Every action the SW dispatches is preceded by an internal check:

1. `scope.exp > now` and `scope.actions_used < max_actions`
2. `command.action ∈ scope.actions`
3. tab origin ∈ `scope.domains`
4. if action ∈ `require_user_confirmation_for`, SW sends a `scope_warning(kind: "confirmation_required")` event and refuses to dispatch until ThunderGate returns a fresh per-action confirmation token (Michael taps Confirm in ThunderCommo within 30s)

If any check fails, SW returns `error { code: "SCOPE_DENIED", retriable: false }`.

### 4.4 Audit record

Each completed action produces an Action Record:

```
{
  action_id, run_id, scope_id, scope_jti,
  action: "click",
  params_redacted: { ref: "el#12", expect_navigation: true },
  dom_before_hash: "sha256:...",
  dom_after_hash:  "sha256:...",
  url_before, url_after,
  ts_dispatched, ts_completed,
  device_id, extension_version,
  signature: "<Ed25519 over the canonicalized record by the device key>"
}
```

These chain: each record carries `prev_hash = sha256(prior record)`. Chain head is anchored at audit-flush time. The chain is the court-admissible artifact: "Jon performed action X at time T under scope S authorized by Michael with intent I, on a page that hashed to H1 before and H2 after, signed by device D."

ThunderGate stores the records in a new `action_audit` table in `context.db` (single source per Design Principle 1). The chain is anchored every 60s by re-signing the head with the gateway key and writing to a separate `audit_anchors` table.

### 4.5 Mid-task expiry

- Scope_warning at T-60s triggers a non-blocking ThunderCommo push: "Jon's AA scope expires in 60s. Extend?"
- At T-0: SW auto-pauses the run, sets popup to red "scope expired", refuses all further actions. Run is not aborted — it's parked.
- ThunderGate may issue a fresh token with same `run_id`; on receipt the SW resumes from `last_action_id`.

### 4.6 Revocation

Michael can revoke from ThunderCommo. ThunderGate emits a `scope_revoked` event over WSS. The SW immediately aborts the current run, clears scope from storage, and acks. Revocation also includes an out-of-band mechanism: if Michael's ThunderCommo says "kill Jon's browser session" and WSS happens to be down, ThunderGate sets a revoke flag in the next handshake — the SW checks this on every reconnect.

---

## 5. AA Portal Automation

The brief calls out aa.com as the immediate target. The portal has known landmines: stateful Travel Planner with no shareable URLs, password expiry interstitials, the "Confirm trip" precision click, and aggressive session timeouts.

### 5.1 State model

ThunderBrowser models AA's portal as an explicit finite state machine. Jon's reasoning loop walks states; the extension confirms each state by DOM evidence before acting.

```
[unauth]    ──login event─▶    [dashboard]
[dashboard] ──open travel──▶   [travel_planner_empty]
[planner_empty] ─search──▶     [search_loading] ─▶ [search_results]
[search_results] ─select──▶    [fare_selection]
[fare_selection] ─continue─▶   [passenger_info]
[passenger_info] ─continue─▶   [payment]
[payment] ─continue──────▶     [confirm_review]
[confirm_review] ─confirm──▶   [confirmed]

Anywhere ─session timeout─▶    [unauth]
Anywhere ─password expired─▶   [password_change]
Anywhere ─error──────────▶     [error]
Anywhere ─captcha────────▶     [captcha_blocked]
```

Each state has:
- **entry detector**: a CSS/text/url pattern with confidence score
- **expected actions**: what Jon is allowed to do here under scope
- **exit transitions**: keyed by Jon's chosen action plus expected DOM signature change

State detection is done in the content script via a small ruleset shipped with the extension (versioned, hot-updateable via a `state_pack` config push from TG). The CS reports `state_detected` events on every meaningful DOM change.

### 5.2 Specific issues

**Login session detection** — `is_logged_in("aa.com")` returns true iff:
- session cookie present (name only, value never read), AND
- `document.body` contains AAdvantage account anchor (`a[href*="aadvantage"]` with non-empty text), OR
- a `data-aa-user` attribute is present on a known landmark

If either is missing → `logged_in: false`. Jon then asks Michael to log in interactively (extension does NOT fill credentials in v1 — see §7).

**Travel Planner navigation** — no direct URLs. Jon issues `click` against the nav element (located by role + accessible name "Travel planner"), then `wait_for_load` with `network_idle`. State detector confirms `travel_planner_empty`.

**The "Confirm trip" precision click** — this is the one we cannot get wrong. Special handling:
1. Before click: full DOM snapshot, snapshot hash recorded in audit.
2. The button is located by role + accessible name + a stability check: same `ref` returned by two queries 250ms apart.
3. The action requires per-action user confirmation (`require_user_confirmation_for` triggers — Michael taps Confirm in ThunderCommo).
4. After click: capture `dom_after_hash`, URL change, and a screenshot stored as evidence (separate `chrome.tabs.captureVisibleTab` call in MV3 — image hash only goes to audit, image bytes stay local).
5. If post-click state is not `confirmed` within 15s: emit `error_detected` and pause the run.

**Password expiry interstitial** — detector watches for:
- URL containing `/password/change` OR `/security/password`
- DOM text matching `/your password (has |is )?expired/i`

On hit: emit `password_expired` event, auto-pause, push to ThunderCommo: "AA password expired. Need Michael at the keyboard."

**Popups and modals** — `detect_modal` runs after every navigation and on any mutation matching modal heuristics. Auto-dismissed: cookie banners (only if classifier confidence > 0.9 and dismissal button text matches allowlist). All others: surface to Jon, do not auto-dismiss.

**Session timeout** — Detector watches for the AA-specific "Your session has timed out" modal and 302→login redirects. On hit: same as `password_expired` — pause, alert Michael, never auto-reauth.

**CAPTCHA** — if any element matches CAPTCHA fingerprints (`iframe[src*="recaptcha"]`, `iframe[src*="hcaptcha"]`, AA's own challenge), emit `captcha_blocked` and pause. Do not attempt to solve. The whole reason the extension exists is to avoid being a bot — if AA decides Michael's browser is a bot, the right move is to let Michael handle it, log the event, and learn.

### 5.3 What ThunderBrowser does NOT do for AA in v1

- **No credential entry.** Michael logs in himself the first time. The extension uses his existing session. v2 may reconsider with a hardware-key gated path.
- **No payment field filling.** Even in v2 this requires a separate authority tier.
- **No automatic confirm.** Every confirm is per-action Michael-tap-confirmed.

These are deliberate. ThunderBrowser is not "Jon books your flight unattended"; it's "Jon walks the form for you, you confirm the irrevocable steps."

---

## 6. Safari Web Extension — Handoff to Mack

### 6.1 Conversion path

Apple ships `safari-web-extension-converter` (Xcode CLI tool). The path:

1. Build Chrome MV3 extension to `dist/chrome/`.
2. `xcrun safari-web-extension-converter dist/chrome --project-location ios-mac/ThunderBrowserSafari --bundle-identifier com.thrustnthunder.thunderbrowser --swift`
3. Resulting Xcode project contains:
   - macOS app shell (required — Safari extensions ship inside a host app)
   - iOS app shell (for Safari iOS extension)
   - Resources/ symlinked to converted manifest + content scripts + SW
4. Manual edits Mack will need to make every time:
   - Update `Info.plist` `NSExtensionPrincipalClass` if we add native messaging
   - Adjust `manifest.json` Safari-only keys (`browser_specific_settings`)
   - Code-sign with Apple Developer ID + provisioning profile

### 6.2 API differences and shims

We will ship a tiny `platform.ts` module so the rest of the code uses `platform.runtime`, `platform.tabs`, etc., and the shim points to `chrome.*` or `browser.*` depending on the build target. `webextension-polyfill` covers ~95% of the gap; the rest:

| Concern | Chrome | Safari | Strategy |
|---|---|---|---|
| Namespace | `chrome.*` callback or promise | `browser.*` promise | webextension-polyfill |
| Service worker | MV3 SW, ephemeral | Persistent background page (Safari MV3 has SW but with quirks) | Code to ephemeral; tolerate persistent |
| `chrome.alarms` | Full support | Supported; minimum period clamps higher | Use 30s for portability |
| `chrome.scripting.executeScript({world})` | ISOLATED/MAIN | Only ISOLATED reliable | Stay ISOLATED (already required) |
| `chrome.declarativeNetRequest` | Full | Limited rule count; some action types missing | Network intercept Phase 2; design assuming Safari limits |
| `chrome.cookies` | Full | Restricted to extension's allowed sites; user must approve site-by-site | We only read presence/expiry, no values — works on both |
| `chrome.tabs.captureVisibleTab` | Yes | Yes, but on iOS Safari may prompt | Hash-only audit pattern unchanged |
| Storage limits | 10MB session, 100MB IndexedDB | Tighter; varies | Audit rotation policy (§7.6) |
| WSS in SW | Works | Works in macOS Safari; in iOS Safari background lifetime is shorter | Same reconnect/replay design absorbs this |

### 6.3 What likely degrades on Safari

- **iOS Safari**: background extension lifetime is unforgiving. Long runs require the host app to be foregrounded. Acceptable — first Safari target is macOS, iOS later.
- **Network interception**: declarativeNetRequest rule budgets are smaller; design Phase 2 with that ceiling in mind (one intercept at a time on Safari, multi on Chrome).
- **Permissions UI**: Safari prompts per-site even with broad host_permissions. Onboarding will need a "click through these prompts" flow in the options page.

### 6.4 Handoff package to Mack

When Chrome MVP is green, deliver:

1. The repo (Chrome MV3 source) at a tagged commit.
2. A `SAFARI_HANDOFF.md` listing every chrome.* API used and which polyfill version covers it.
3. A capability matrix marking which actions in §3 have been tested on Safari and which have not.
4. A test WSS endpoint pointing to a Safari-specific TG instance so Mack can exercise actions without touching prod.
5. The fixture site (§8.3) bundled so Mack can run scenarios offline.
6. A signed test scope token good for the dev TG, for end-to-end verification.

Mack's lane starts at the `safari-web-extension-converter` invocation and ends at TestFlight on macOS. CLI Jon stays on the Chrome side and the bridge.

---

## 7. Security Model

The extension can read and act on every page in Michael's browser. That is the entire threat surface. Compromise of either side (extension code or ThunderGate) must not exfiltrate Michael's session.

### 7.1 Authentication: extension ↔ ThunderGate

- **Pairing (once)**: Michael opens the options page; it generates an Ed25519 keypair via `crypto.subtle.generateKey({name:"Ed25519"}, false, ["sign"])` — non-extractable. The public key plus a one-time pairing code is presented as a QR. Michael scans on his phone (ThunderCommo) which sends it to ThunderGate. ThunderGate records `{ extension_pair_id, device_pubkey, paired_at, user }` in `context.db`.
- **Every WSS connect**: SW generates a short-lived (5 min) JWT signed by the device key. Server validates against pinned device pubkey. No long-lived bearer token sits in storage that, if stolen, grants WSS access.
- **Key rotation**: SW rotates device key every 90 days; rotation messages are signed by the old key and pushed in a `device_rotation` event before old key retires.
- **No password, no shared secret.** The device key is the credential.

### 7.2 Defense against compromised ThunderGate

If ThunderGate is compromised, what stops it from instructing the extension to exfiltrate Michael's bank cookies?

1. **Domain allowlist enforced inside the extension.** Hardcoded base (`aa.com`, `*.aa.com`); extensible via the options page; never extensible via WSS push. Commands targeting non-allowlisted origins return `NOT_ALLOWLISTED` immediately. ThunderGate cannot expand the allowlist remotely.
2. **No raw cookie read.** §3.5 — extension returns only presence and expiry, never values. There is no command that returns a cookie value over WSS. If a future feature requires it, the design must change here and be re-reviewed.
3. **No password field read.** Content script's redactor (§7.3) blanks any `input[type=password]` and any field with `autocomplete="cc-*"`. `fill secret: true` flows write-only.
4. **Per-action confirmation gate.** Irrevocable actions (purchases, confirmations, anything ThunderGate marks as `require_user_confirmation_for`) require Michael's ThunderCommo tap. A compromised TG cannot bypass this — confirmation tokens are signed by Michael's phone key, not TG's.
5. **Action budget.** `max_actions` caps how much damage a runaway scope can do before re-grant. Default cap: 200 per scope.
6. **Audit chain.** Every action is signed by the device key. A compromised TG can issue malicious commands but cannot forge the device's signature on the action record. Forensic recovery is possible.

### 7.3 Input redaction (content script)

Before any DOM snapshot leaves the content script:

- `input[type=password]` → value blanked, replaced by `"[REDACTED:password]"`
- `input[autocomplete^="cc-"]`, `input[name*="card"]`, `input[name*="cvv"]` → blanked
- `input[autocomplete*="ssn"]`, regex-detected SSNs/credit-card patterns in any text node → masked
- Cookie/storage values: never read by snapshot at all
- A configurable regex denylist (extensible via options page only) for site-specific gotchas

The redactor runs in the content script, *before* the data crosses to the SW. The SW never sees plaintext secrets; ThunderGate definitely doesn't.

### 7.4 Domain allowlist behavior

- Default-deny. Empty allowlist on first install.
- Pairing flow seeds the allowlist with `aa.com` and the user can add more via options.
- The content script declares matches via manifest `host_permissions`. The SW *also* enforces in code — defense in depth. A command for a tab whose origin doesn't match the allowlist returns `NOT_ALLOWLISTED` before any CS message is sent.
- "All sites" is not a setting we expose. If Michael wants it, he uses Chrome's developer console — not the extension UI.

### 7.5 Kill switch

- Popup has a big red "Pause All" button. Sets `paused=true` in storage. SW refuses all commands until cleared.
- ThunderCommo has the same. Pushes `pause_all` event to extension.
- An out-of-band cutoff: if Michael uninstalls the extension or removes the pair, ThunderGate's per-extension queue is dumped and the device pubkey is marked revoked.

### 7.6 Audit log handling

- Local audit log in IndexedDB. Capped at 10MB; rotation policy keeps last 1000 actions or 7 days, whichever is greater.
- Server-side audit in `context.db` is the authoritative copy; local is a buffer.
- Audit log viewer in options page shows every action with diff hashes and outcome — Michael's transparency surface.

### 7.7 Content-script isolation

- All injected scripts run in ISOLATED world. Page-side scripts cannot see, intercept, or impersonate them.
- The extension never injects `<script>` tags into the page DOM. Anything that would require MAIN world execution is rejected at design time.
- CSP-compatible: extension makes no use of `eval`, `new Function`, or inline scripts.

### 7.8 Supply chain

- Extension is built from a pinned git tag. CI produces a reproducible bundle. The build artifact's SHA-256 is recorded in `context.db` at pairing time; the SW reports its own bundle hash on every `ready` event and ThunderGate rejects mismatches.
- Dependencies: zero non-essential. webextension-polyfill (vendored, pinned hash). No analytics, no telemetry SDKs.

---

## 8. Development + Testing Path

### 8.1 Two extension identities

| Profile | ID | TG endpoint | Use |
|---|---|---|---|
| **dev** | unpacked, ID derived from key.pem in repo | `wss://gateway-dev/browser` | Developer machine, breaking changes |
| **prod** | signed/published, stable ID | `wss://gateway/browser` | Michael's daily browser |

The dev extension's `manifest.json` has `name: "ThunderBrowser (dev)"`, a different icon, and a yellow popup background — visually impossible to confuse with prod. The SW refuses to talk to prod TG; the prod SW refuses to talk to dev TG. Mismatch is rejected at handshake.

### 8.2 Test Chrome profile

```
/Users/<dev>/Library/Application Support/Google/Chrome ThunderDev/
```

Distinct profile means: separate cookies, separate password store, separate everything. Developer cannot accidentally run automation against Michael's real AA login. The launcher script:

```
open -na "Google Chrome" --args \
  --user-data-dir="$HOME/Library/Application Support/Google/Chrome ThunderDev" \
  --load-extension="$(pwd)/dist/chrome"
```

### 8.3 Fixture site

Recreate AA's Travel Planner states as static HTML + minimal JS, served by a local Express server on `http://localhost:7860`. Fixtures cover:

- `/login` → unauth
- `/dashboard` → dashboard
- `/travel-planner` → planner_empty
- `/travel-planner/results` → search_results (3 variants: normal, sold-out, error)
- `/travel-planner/fare` → fare_selection
- `/travel-planner/passenger` → passenger_info
- `/travel-planner/payment` → payment
- `/travel-planner/confirm` → confirm_review (with the precision-click button)
- `/travel-planner/confirmed` → confirmed
- `/password-expired` → password_change interstitial
- `/captcha` → captcha_blocked variant
- `/timeout` → session timeout modal

The state detector ruleset is validated against the fixture site as part of CI.

### 8.4 Mock ThunderGate

A separate node script `tg-mock` exposes the WSS endpoint and replays canned command streams from `*.tbscript` files:

```
scripts/
  aa_happy_path.tbscript
  aa_sold_out_replan.tbscript
  aa_password_expired.tbscript
  aa_captcha_dead_end.tbscript
  aa_session_timeout_midflight.tbscript
```

Each script is a sequence of commands plus assertions on the events that come back. The runner reports pass/fail per scenario. Mock TG can also be put in "interactive" mode where a developer drives commands by hand from a REPL.

### 8.5 Recording mode

When `record=true` is set on a run, the extension dumps every action + DOM-before/after to a local `.tbrec` file. A separate `tbrec replay` tool feeds the file to the fixture site and the extension and verifies the same diffs come out. This is the regression harness — equivalent role to Ghost Jon for the runtime.

### 8.6 Real-AA dry run protocol

Once fixtures are green, the path to real AA:

1. **Dry-run mode**: Michael is at the keyboard. Jon issues commands. Extension shows every action in the popup with a "proceed" button. Michael clicks proceed for each. No auto-progression. Effectively step-by-step.
2. **Shadow mode**: Michael drives the booking manually. Extension watches and records what Michael does. Records become the canonical happy-path script for that scenario.
3. **Assisted mode**: Jon proposes the next action; Michael accepts/rejects. Faster than dry-run, slower than auto.
4. **Auto mode (per-action confirm only)**: Jon drives; Michael confirms only the irrevocable steps. This is the v1 ceiling for real AA.

### 8.7 Doctor coverage

ThunderBrowser-specific Doctor checks (extend the existing daily-health-check.sh or its successor):

- WSS reconnect success rate > 99% over 24h
- Action audit chain unbroken (no missing prev_hash links)
- Fixture suite pass rate = 100% on every deploy
- No `SCOPE_DENIED` events from prod that don't have a corresponding ThunderCommo decline (would indicate clock skew or token corruption)
- Bundle hash matches expected pinned hash
- No `cs_disconnected` events lasting > 5s in last 24h

---

## 9. MVP — What to Build First

Six phases. Each one ships independently; everything after Phase 1 is gated on the prior phase being Doctor-green.

### Phase 0 — Skeleton (target: week 1)

- Manifest V3 scaffold, popup, options page, IndexedDB schema
- `platform.ts` shim, webextension-polyfill vendored
- Background SW: WSS connect to mock TG, heartbeat via `chrome.alarms`, reconnect on wake
- Pairing flow: keypair gen, QR display, pair record written to dev TG
- **Exit criteria**: dev extension installs in a fresh profile, pairs, connects, heartbeats survive SW restart

### Phase 1 — Action API + AA fixtures (week 2–3)

- Content script bus, message envelope, isolated-world injection
- Actions: `navigate`, `click`, `fill` (no secret), `select`, `check`, `scroll_to`, `wait_for_load`, `snapshot_dom` (structured + accessibility), `query`, `get_url`, `detect_modal`, `detect_error`, `is_logged_in`
- State detector for AA fixture pages
- `tg-mock` runner with five fixture scripts (happy path, sold out, password expired, session timeout, captcha)
- **Exit criteria**: all five fixture scripts pass green in CI; recording mode produces replayable `.tbrec` files

### Phase 2 — BYOAA authority (week 4)

- Scope token verify (Ed25519 against pinned kid → pubkey)
- Per-action gates: domain, action allowed, action budget, expiry, per-action confirmation requirement
- Audit record creation, device signing, chained hashes, IndexedDB store, WSS flush + server ack
- Scope expiry/extend/revoke handling
- **Exit criteria**: invalid scope token rejected; mid-task expiry produces clean pause + extend cycle; revoke aborts within 200ms

### Phase 3 — Real AA dry run (week 5)

- Wire to prod TG endpoint with prod scope-signing key pinned
- Dry-run mode (manual confirm every action)
- Assisted mode
- Doctor checks live
- Michael in seat for first real run, target: AA flight status check (read-only — no booking)
- **Exit criteria**: ten successful read-only dry runs against real AA over five days; zero scope violations; zero unredacted secret-field leaks (audit verified)

### Phase 4 — Auto with per-action confirm (week 6)

- Auto mode enabled, irrevocable actions gated by ThunderCommo confirmation
- First real auto run: jumpseat MIA→DFW scenario, Michael in seat with veto power
- Iterate on state detector based on real-AA DOM patterns
- **Exit criteria**: first end-to-end jumpseat reservation completed by the extension; Michael confirms each irrevocable step; full audit chain inspected and verified

### Phase 5 — Safari handoff (week 7+)

- Freeze Chrome feature surface
- `SAFARI_HANDOFF.md` written
- `xcrun safari-web-extension-converter` run; project handed to Mack
- Mack drives macOS Safari; CLI Jon stays on Chrome and bridge

### Phase 6 — Network intercept + multi-tab (future)

- `install_intercept` / `read_intercepted` for AA's XHR endpoints
- Cross-tab orchestration (Jon coordinates booking on one tab while reading status on another)
- These are deferred; the MVP does not need them.

### MVP cut line

Phases 0–2 are the minimum viable extension. Phase 3 is the minimum useful extension. Anything past Phase 4 is scale-up.

Explicitly deferred:
- Multi-user (Michael-only until BYOAA-on-Android lands)
- Mobile Safari (iOS) — macOS first
- Generic recorders for arbitrary sites (we are AA-first)
- LLM running inside the extension — reasoning stays on TG
- Headed Playwright fallback — abandoned by design

---

## 10. Open Questions / Followups

1. **Action manifest format alignment.** BYOAA's Action Manifest spec (see `BYOAA_CLI_JON_ANALYSIS.md` — still undefined) needs to match the scope-token `permissions` shape here. If Alex's Action Manifest lands later, we may need to adapt §4.1.
2. **Loop / Clearing House for financial transactions.** Out of scope for AA jumpseat (no money moves). When ThunderBrowser is asked to do anything that touches payment (Amazon, domain registration, etc.), the design needs a second authority tier — likely Loop's Clearing House per Alex's hint in `THUNDERBROWSER.md`. Defer until Phase 6+.
3. **Cookie-presence vs cookie-content tradeoff.** Some sites won't be diagnosable by presence alone. If we need to read a session-value hash (not the value) to disambiguate, that's a new command and a new threat-model review.
4. **Patent prep.** Per `THUNDERBROWSER.md` §"Invention Attribution" — keep this design private until provisional filing. The KYA-scoped action-record chain in §4 is the patent-relevant novelty; do not discuss publicly.
5. **Ghost-style shadow eval for ThunderBrowser.** Phase 1's recording mode is the seed for this. Eventually we want a Doctor-green metric for "extension behaved correctly vs a recorded baseline" the way Ghost Jon scores Haiku vs OpenClaw. Not MVP — but plan for it in the audit schema.

---

*CLI Jon | ThunderBase | 2026-05-11*
