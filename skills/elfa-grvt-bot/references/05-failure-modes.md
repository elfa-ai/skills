# 05 - Failure modes

For every failure the bot can encounter, what to detect, what to do, and what alert (if any) to emit. Read this together with `03-state.md` (alert categories) and `04-algorithms.md` (where each branch is called from).

The matrix is the authoritative list. Anything not in the matrix is undefined behavior; surface it as a generic `error / unexpected_exception` alert with the exception class and message.

## During strategy authoring (agent session, hitting Elfa)

| Trigger | Detection | Action | Alert |
|---|---|---|---|
| `ELFA_API_KEY` rejected | `POST /v2/auto/chat` returns 401 | Stop the authoring flow. Tell the user to recheck the key. | none (this is a user-facing error in the agent's response, not a registry alert) |
| Builder Chat returns malformed EQL (no fenced JSON, or unparseable) | Parse fails | Re-prompt Builder Chat with `sessionId` set, or ask the user to rephrase. Do NOT hand-edit the JSON. | none |
| Validate returns `valid: false` | Response `errors` array non-empty | Surface the errors verbatim to the user. Stop. | none |
| Validate returns `valid: true` but the EQL doesn't match user intent | Manual review during plan-confirmation step | Re-prompt or rephrase. | none |
| GRVT does not list the requested symbol | `instruments` cache check fails for that symbol | Tell user "GRVT doesn't have that token". Stop. | none |
| User says no at the confirmation step | User input is not "yes" | Stop. Do NOT call `create_query`. | none |

## During receiver startup

| Trigger | Detection | Action | Alert |
|---|---|---|---|
| Required env var missing | Config validation | Refuse to start. Print the missing var name. Exit 1. | none (logged to stderr) |
| `GRVT_ENV != prod` | Config validation | Refuse to start. Print the rejection. Exit 1. | none |
| GRVT login returns 200 with no cookie | `_load_instruments_cache` -> first auth call | Refuse to start. Surface response body (look for `error` field). Exit 1. | none (preflight catches this earlier; if it didn't, refuse loudly) |
| GRVT login returns 401 | Auth call | Refuse to start. | none |
| Network unreachable | Auth or instruments call | Refuse to start with a clear "network error" message. | none |

## During SSE consumption

| Trigger | Detection | Action | Alert |
|---|---|---|---|
| `event:` is `notification` or `query.triggered`, all required fields present, `status == 'triggered'`, `executionId` matches local strategy | Parser yields event | Call `process_fire` | (downstream alerts from `process_fire`) |
| Frame missing a required field | `build_event` returns None | Drop frame, log WARN with the missing fields | If 3+ drops in 10 min for same query: `error / parser_drift` |
| `data:` is not valid JSON | `json.loads` raises | Drop frame, log WARN with first 200 chars of `data` | Same as above |
| `status != 'triggered'` | `build_event` check | Drop frame, log WARN with the actual status | none (not unusual; could be `pending`, `evaluating`, etc.) |
| `queryId` in payload != stream URL's query id | `build_event` check | Drop frame, log WARN | If repeated: `error / parser_drift` |
| `event:` is not in the accepted set | `build_event` check | Drop silently (no log) | none |
| Stream returns 401 | `stream_notifications` raises `ElfaStreamError(401, ...)` | Set strategy to `failed`, exit task | `error / strategy_terminated_remotely` |
| Stream returns 404 | Same | Set strategy to `failed`, exit task | `error / strategy_terminated_remotely` |
| Stream returns 410 | `stream_notifications` exits cleanly (empty stream) | Loop back to poll-query for reconciliation | none |
| Stream returns 5xx | `ElfaStreamError(5xx, ...)` | Exponential backoff (2s -> 60s), reconnect | none |
| Stream returns 204 | Empty stream | Loop back to poll-query | none |
| Connection drops mid-stream (no error, just EOF) | Async iteration ends | Loop back to poll-query, no backoff (reset to initial) | none |
| Keep-alive comment received | Line starts with `:` | Skip, continue iterating | none |

## During fire processing

| Trigger | Detection | Action | Alert |
|---|---|---|---|
| `executionId` already in `fires` table | `insert_fire_marker` raises IntegrityError | Skip silently; log "duplicate event {id}, skipped" | none |
| No local strategy for `query_id` | `registry.get_strategy` returns None | Update fire outcome to `unknown_strategy` | `warning / unknown_strategy` |
| GRVT `fetch_mid_price` fails (network, auth, no quote) | Raises `GrvtError` | Update fire outcome to `grvt_error`, set strategy to `fired` | `error / grvt_other` |
| Guardrails reject (notional cap, env mismatch, etc.) | `check_guardrails` returns `Reject(reason)` | Update fire outcome to `rejected_guardrail`, set strategy to `fired` | `warning / guardrail_rejected` with the reason |
| `set_leverage` fails | Raises `GrvtError` | Continue with order placement anyway; surface as warning | `warning / grvt_set_leverage` |
| Order placement returns insufficient margin | GRVT response code maps to `InsufficientMargin` | Update outcome to `grvt_error`, set strategy `fired` | `error / insufficient_margin` |
| Order placement returns auth failure | Response body contains `authenticate` or 401 | Same | `error / grvt_auth_failed` |
| Order placement returns any other error | Raises `GrvtError` | Same | `error / grvt_other` |
| Order placed successfully | Bulk submit returns AND parent is confirmed accepted/filled by `client_order_id` via order history/open orders/position check | Update outcome to `placed` with order ids/client ids, set strategy `fired` | `info / order_placed` |
| Uncaught exception inside `process_fire` | Top-level try/except | Update outcome to `unknown` with traceback, set strategy `fired` | `error / unexpected_exception` |

## During poll-query reconciliation

| Trigger | Detection | Action | Alert |
|---|---|---|---|
| Remote status is `active` and no new executions | Normal case | Continue (open SSE) | none |
| Remote status is `active` but executions array has rows not in local `fires` | Receiver was offline during a fire | Insert fire marker, emit alert, continue (SSE will get any new fires) | `error / manual_intervention_required` per missed execution |
| Remote status is `triggered` and we already have the execution locally as `placed` | Live SSE got the fire first | Sync local status to `fired` | none (no double-alert) |
| Remote status is `triggered` and execution is not locally tracked | Offline fire | Sync local status to `fired`, insert marker | `error / manual_intervention_required` |
| Remote status is `expired` | Time-based termination | Sync local status to `expired` | `info / strategy_terminated_remotely` |
| Remote status is `cancelled` | User cancelled via Elfa API directly | Sync local status to `cancelled` | `warning / strategy_terminated_remotely` |
| Remote status is `failed` | Server-side failure | Sync local status to `failed` | `error / strategy_terminated_remotely` |
| Remote status is `recurring` | Bot does not support recurring | Sync local status to `failed` | `error / strategy_terminated_remotely` with note that recurring is unsupported |
| Poll-query returns 404 | Query deleted on Elfa | Sync local status to `failed` | `error / strategy_terminated_remotely` |
| Poll-query returns 5xx | Transient | Log, retry on next supervisor cycle | none |

## During preflight

| Trigger | Detection | Action | Alert |
|---|---|---|---|
| `ELFA_API_KEY` missing | Env check | Exit 1 with clear message | none |
| `GET /v2/auto/queries?limit=1` returns 401 | HTTP status | Exit 1 with "ELFA_API_KEY rejected" | none |
| `GET /v2/auto/queries?limit=1` returns 5xx or network error | Exception or status | Exit 1 with the underlying message | none |
| GRVT login returns 200 + cookie present | Normal | Mark `grvt: ok` | none |
| GRVT login returns 200 + no cookie + body contains "location" | Geo-block | Exit 1 with the body's `error` field and a "GEO-BLOCK" prefix | none |
| GRVT login returns 200 + no cookie + body contains "invalid api key" | Bad key | Exit 1 with the body's error | none |
| Telegram configured (both vars set) and `getMe` returns 401 | Bad token | Exit 1 | none |
| Telegram configured and `getMe` returns 200 but `ok: false` | Other Telegram problem | Exit 1 with the body | none |
| Only one of TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID set | Env check | Exit 1; tell user to set both or leave both blank | none |
| Telegram not configured (neither set) | Env check | Skip Telegram probe; print "(not configured)" | none |

Preflight failures NEVER write alerts to the registry: they exit non-zero before the receiver starts, so there's no running session to surface alerts to. The user reads the exit message.

## Alerts that must be raised but have no obvious trigger above

Some alerts are not driven by a specific failure but by sustained conditions. Implement these as periodic checks (every minute or so):

| Trigger | Detection | Action | Alert |
|---|---|---|---|
| Receiver hasn't logged anything in 15 min | Watchdog timer | Heartbeat alert | `warning / receiver_silent` (debounced; only emit once until normal traffic resumes) |
| Alerts table has > 50 unacked rows | Periodic check | Possibly the user stopped acking; warn but don't block | `warning / alerts_backlog` (debounced) |

These are optional for v1. Skip them in the initial implementation if you want to keep scope tight.

## Recovery strategies

For each error class, the bot's recovery policy:

- **Transient (network, 5xx)**: exponential backoff in the strategy loop. Receiver process stays up. No alert unless backoff exceeds some threshold (e.g., 1 hour cumulative); optional `warning / persistent_network_error`.
- **Terminal remote (404, query gone, status `failed`)**: sync local status, alert, exit the per-strategy task. Supervisor reaps it.
- **Auth (401)**: exit the per-strategy task with `failed`, alert. Don't try to refresh credentials (the receiver was started with bad credentials, so the whole process is suspect). User must restart after fixing `.env`.
- **GRVT-specific (insufficient margin, geo-block at order time, etc.)**: alert and stop trying for that strategy. The strategy transitions to `fired` (even though no order was placed); the user reviews manually.
- **Parser drift**: alert, drop the frame, continue. The supervisor keeps trying. If the drift is permanent, the user will see `manual_intervention_required` alerts from poll-query reconciliation as the only signal that fires are happening.

The general principle: **fail-closed at the order-placement boundary, fail-open at the SSE-consumption boundary**. The receiver should keep running as much as possible (so future strategies can fire), but should refuse to place orders on partial or ambiguous data.
