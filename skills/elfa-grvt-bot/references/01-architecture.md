# 01 - Architecture

The bot is two surfaces wired together by a local SQLite registry. Read this whole file before reading any other reference; downstream specs assume the shapes described here.

## The two surfaces

**Authoring** is synchronous, agent-driven, happens inside a chat session:

```
user describes strategy in chat
    -> agent prepends "Notify me when: "
    -> agent POSTs /v2/auto/chat (Builder Chat)
    -> agent extracts EQL JSON block from response.response
    -> agent validates GRVT side: symbol exists? size sane?
    -> agent POSTs /v2/auto/queries/validate
    -> agent shows full plan, waits for explicit "yes"
    -> agent POSTs /v2/auto/queries (creates the active query)
    -> agent INSERTs a row into the local registry mapping query_id -> order spec
```

After authoring, the chat session is no longer required. The next surface picks it up.

**Execution** is asynchronous, long-running, runs as a separate OS process:

```
supervisor (polling local registry every 5s):
    -> for each strategy with status='active' and no live task:
       spawn an async task -> strategy_loop(query_id)

strategy_loop(query_id):
    -> GET /v2/auto/queries/<id>  (poll-query for status reconciliation)
       -> if remote status is terminal: sync local status, emit alert, exit
    -> GET /v2/auto/queries/<id>/stream  (SSE connection)
       -> for each well-formed trigger frame:
          - dedupe by executionId
          - run guardrails (max_notional, symbol, etc.)
          - fetch current mark/mid price from GRVT
          - submit one atomic OTOCO order (entry + TP + SL)
          - mark fire outcome, transition strategy to 'fired', exit
    -> on stream close without fire: poll-query again; if terminal, sync + exit
    -> on transient error: exponential backoff (start 2s, cap 60s), reconnect
```

The supervisor also cancels per-strategy tasks when their local status leaves `active` (CLI cancel, terminal sync, etc.).

## Single-fire by design

Each strategy fires at most once and transitions to a terminal local state (`fired`, `expired`, `cancelled`, `failed`). The bot does not maintain open positions, does not re-arm, does not support `recurring` Elfa queries. Recurring queries get mapped to local `failed` and surfaced as an alert.

This constraint simplifies the dedupe story enormously: once a strategy has any row in the `fires` table, all subsequent SSE frames and poll-query observations for that strategy are treated as duplicates.

## No inbound HTTP (do not build a webhook receiver)

The receiver only opens outbound connections (Elfa SSE, GRVT API, Telegram). No bind, no public URL, no tunnel, no webhook endpoint. The bot runs on a laptop, in a Docker container with no port mapping, behind NAT, anywhere.

This eliminates the unauthenticated-remote-trigger risk class that the previous webhook-based architecture had: trigger source is Elfa itself, authenticated by the bot's own outbound API key.

The implementing agent MUST NOT introduce:

- An HTTP server library (`flask`, `fastapi`, `aiohttp.web`, `starlette`, etc.)
- A listening socket
- A public URL or tunnel (cloudflared, ngrok, similar)
- A Webhook delivery action in Elfa (the bot only creates notify-style queries; trigger delivery is the SSE stream)

If a future requirement seems to need inbound delivery, the answer is to add a new notify-style channel to Elfa's actions and consume it through the existing SSE path, NOT to open a port.

## Persistence boundaries

| What | Where | Why |
|---|---|---|
| Strategy metadata + order spec | SQLite `strategies` table | Survives restart, shared between authoring + receiver |
| Fire idempotency | SQLite `fires` table | Survives restart, dedupe across SSE + poll-query |
| Alerts | SQLite `alerts` table | Survives restart, surfaced to agent on next session |
| GRVT auth state (cookies, account_id) | In-process only | Refreshed on each receiver startup |
| Per-strategy SSE task state | In-process only | If the receiver crashes mid-strategy, the next start reconciles via poll-query |

Trade execution state lives on GRVT, not locally. The bot does not persist positions, fills, or PnL. Anything position-related is fetched live from GRVT (`fetch_positions`, `fetch_balance`, `fetch_open_orders`) when needed.

## The two ID namespaces (only one matters now)

Elfa returns three kinds of identifiers and the bot must distinguish them:

| Identifier | Where | Format | Use |
|---|---|---|---|
| `queryId` | `/v2/auto/queries` response (as `id`), all subsequent endpoints | UUID | Primary key for strategies. Maps to GRVT order spec. |
| `executionId` | SSE payload `data.executionId`, poll-query `executions[i].id` | UUID, same namespace across both channels | **Canonical idempotency key for fires.** Primary key in the `fires` table. |
| SSE `id:` line | SSE wire protocol level | UUID, different from above | Not used by the bot. Read and discarded. |

Earlier versions of the spec described an `evt_xxx` vs `exec_xxx` namespace split between SSE and poll-query. Production does not implement that split: SSE `executionId` and poll-query `executions[i].id` are the same UUID for the same fire. Verified 2026-05-13. See `references/02-protocols.md` for the captured frames.

## Alerts as the user-visible channel

The bot does not write user-facing logs. Everything important goes into the `alerts` table and surfaces via:

1. **Agent session pull**: each new agent session reads `SELECT * FROM alerts WHERE acked_at IS NULL` and presents them at the top of the response. The user clears alerts with `ack <id>` or `ack all`.
2. **Telegram push** (optional): if `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` are set, alerts are also pushed to Telegram in real time. If either is missing, push is silently skipped; in-chat alerts still work.

Categories the bot must emit (the matrix in `05-failure-modes.md` is authoritative):

- `trigger_received` (info): SSE delivered a fire; order placement about to start
- `order_placed` (info): OTOCO parent order confirmed accepted or filled on GRVT after submit verification
- `guardrail_rejected` (warning): fire failed a local check (size, symbol, leverage); no order sent
- `grvt_auth_failed` (error): preflight or order placement failed because GRVT auth is broken
- `grvt_other` (error): unspecified GRVT error during order placement
- `insufficient_margin` (error): GRVT rejected the order for margin reasons
- `manual_intervention_required` (error): a fire happened while the receiver was offline; user must reconcile manually
- `strategy_terminated_remotely` (info/warning/error depending on terminal status): Elfa reports the strategy as terminal
- `unknown_strategy` (warning): SSE delivered a fire for a query_id not in local registry
- `parser_drift` (error): SSE frame was dropped because required fields are missing (signals an upstream schema change)

## What runs where

```
+-----------------------------------+    +-----------------------+
|        Agent chat session         |    |  Receiver process     |
|                                   |    |  (python -m ...)      |
|  - reads pending alerts on start  |    |                       |
|  - authors strategies             |    |  - supervisor loop    |
|  - cancels strategies             |    |  - per-strategy tasks |
|  - calls CLI: list / ack / cancel |    |  - SSE streams        |
+----------------+------------------+    +----------+------------+
                 |                                  |
                 v                                  v
              +-----------------------------------------+
              |   Local SQLite (registry.db)            |
              |   tables: strategies, fires, alerts     |
              +-----------------------------------------+
                 |                                  |
                 v                                  v
              Elfa Auto                       GRVT API
              (HTTPS + SSE)              (HTTPS, EIP-712 signed POSTs)
                                              |
                                              v
                                        Telegram Bot API
                                       (optional, push only)
```

Authoring and execution share state only through the SQLite registry. They never communicate directly. This decouples them completely: the receiver can be restarted, moved to a different host, or run on a PaaS, while the agent session continues authoring from a laptop. As long as both point at the same `registry.db` (and the receiver has its own `.env` with the same Elfa/GRVT credentials), they work.

## Deployment shape

**Primary target: local install.** This skill walks the user through running the bot on their own machine: laptop, workstation, dev box. `elfa-grvt-bot bootstrap` installs deps, runs preflight, and starts the receiver in the background. `elfa-grvt-bot teardown` stops it. That is the deployment shape the spec orchestrates end to end.

The bot is environment-agnostic by construction (outbound-only, no public DNS, no inbound ports), so the same code runs unchanged on a VPS, Fly.io, Railway, AWS, etc. **Moving the bot off the user's local machine is out of scope for this skill.** If the user wants that, they (or a different skill) figure it out separately. Helpful pointers for that path:

- Any host that can run a long-running Python process works.
- Mount a persistent volume for `registry.db` (else strategies are lost on redeploy).
- Required outbound HTTPS: `api.elfa.ai`, `*.grvt.io`, `api.telegram.org`.
- Host MUST be in a region not geo-blocked by GRVT (see `10-troubleshooting.md`).
- Webhooks-style PaaS that scales to zero between requests will NOT work. The receiver must stay continuously up to hold SSE connections open.
- The bot supports daemon mode (`elfa-grvt-bot run --daemon`) which writes a PID file and log file; wrap that in systemd / Fly.io's process config / etc. as appropriate to your host.

Trade-offs the user should be aware of when deciding local vs. hosted:

- **Local laptop**: zero cost, full control, but laptop sleep loses SSE connections until reconnect. Fires that happen during sleep land on Elfa's side and surface as `manual_intervention_required` alerts on next start. Fine for small/intermittent strategy sets; risky if the user expects 24/7 coverage.
- **Hosted (VPS / PaaS)**: continuous uptime, automatic restart on failure, predictable region for geo-block avoidance. Costs money, adds an operational surface to maintain.

If the user asks about hosting during install, briefly mention the trade-offs and offer to point them at this file. Do not extend the install walkthrough to cover hosting.
