# 09 - Strategy authoring

The chat flow an agent follows when the user describes a strategy. This applies in any chat-driven environment (Claude Code, Cursor, OpenCode, etc.) where the agent has access to: the Elfa API (via `requests` or equivalent), the local CLI (`elfa-grvt-bot add ...`), and the file edit + bash tools.

The `AGENTS.md` shipped by `init` is the in-project version of this flow; this reference is the canonical source.

## On every new chat session

Before responding to anything else:

1. Read pending alerts: `elfa-grvt-bot alerts --pending`.
2. If there are alerts, surface them at the top of the response in this format:

```
N unacknowledged alert(s):
- **#<id>** [<severity>/<category>] <message> (strategy=<query_id>)

Say `ack <id>` to clear, or `ack all` to clear all.
```

3. If the user says `ack <id>` or `ack all`, run `elfa-grvt-bot ack <arg>`.

If there are no pending alerts, say nothing about alerts and continue.

## When the user describes a strategy

Treat any of these as "the user is describing a strategy":

- "Long X when RSI ..."
- "Notify me when ..."
- "Place a market buy when ..."
- "Short Y if ..."
- "Watch for ..." with a trigger condition

Steps:

### 1. Frame and forward to Builder Chat

Always prepend `Notify me when: ` to the user's description, even if they already wrote "notify me when". (Idempotent prepend: if their text already starts with that exact prefix, don't double-wrap.) Call:

```
POST https://api.elfa.ai/v2/auto/chat
header: x-elfa-api-key
body: {"message": "Notify me when: <user description>"}
```

If you've already called Builder Chat for this conversation, pass `sessionId` from the previous response to preserve context.

### 2. Extract EQL from the response

The response field `response` is markdown text containing the EQL inside a fenced JSON block. Extract verbatim:

```python
import re, json
match = re.search(r"```json\s*(\{.*?\})\s*```", response_markdown, re.S)
eql = json.loads(match.group(1))
```

**Do NOT hand-edit the EQL.** If it doesn't match the user's intent, re-prompt Builder Chat (with `sessionId`) or ask the user to rephrase. The conditions block is Builder Chat's authority.

The `actions` block Builder Chat emits is also passthrough: this bot consumes triggers via SSE on the query id, not via the actions block, so the actions are ignored at runtime. Still pass them through unchanged to `create_query`.

### 3. Collect GRVT order spec

Ask the user for anything they didn't volunteer:

- **Symbol** on GRVT. Verify it exists before continuing. Use the public market-data endpoint:
  ```
  POST https://market-data.grvt.io/full/v1/instruments
  body: {"kind":["PERPETUAL"],"is_active":true,"limit":1000}
  ```
  Find the instrument by `instrument` field (e.g. `BNB_USDT_Perp`). If not found, tell the user "GRVT doesn't have that token" and stop.
- **Size** (in base units). Compute the implied notional at the current mid (from `/full/v1/mini`) and surface it.
- **Order type**: market (default) or limit. If limit, ask for a `price`.
- **Leverage** (optional). If omitted, GRVT uses the account default.
- **Time-in-force** (optional, default GTC).
- **`max_notional_usd` cap** (REQUIRED). This is the safety primitive. Suggest a value based on the user's expressed size, but require explicit confirmation.
- **TP/SL percentages** (optional). If provided, the receiver will compute TP/SL prices from the mid at fire time and submit atomically.

### 4. Validate

```
POST https://api.elfa.ai/v2/auto/queries/validate
body: {"query": <the EQL dict>}
```

If `valid: false`, surface the `errors` array and stop. Do not proceed.

If `valid: true`, note the `estimatedCredits` cost; surface it in the plan for the user.

### 5. Show the plan and wait for explicit "yes"

Format the plan something like:

```
Title:        <Builder Chat's title or user-provided>
Description:  <one or two sentences>

Elfa condition (EQL):
  AND:
    - <condition 1 in human-readable form>
    - <condition 2 ...>

Elfa action (passthrough; bot consumes via SSE):
  notify

Expiry:           <expiresIn>
Estimated cost:   <N> credits

GRVT order spec (placed when SSE fires):
  symbol:           <symbol>
  side:             <buy/sell>
  size:             <amount> <base>  (~$<notional> at mid $<mid>)
  order type:       <market/limit>
  limit price:      <if limit>
  leverage:         <if set>
  TP / SL:          <if set>
  max_notional_usd: <cap>
  env:              prod
```

Wait for the user to reply with an explicit "yes" (or "no", or a correction). Anything other than an unambiguous yes means do NOT proceed.

### 6. Create + register

On approval:

```
POST https://api.elfa.ai/v2/auto/queries
body: {
  "title": <title>,
  "description": <description>,
  "query": <eql dict>
}
```

Response has the strategy in `id`, NOT `queryId` (production reality; see `02-protocols.md`). Capture that id.

Then register locally:

```
elfa-grvt-bot add \
  --query-id <id from create_query response> \
  --title <title> \
  --description <description> \
  --eql-json <serialize the eql dict to JSON> \
  --symbol <symbol> \
  --side <buy/sell> \
  --amount <amount> \
  --order-type <market/limit> \
  --max-notional-usd <cap> \
  [--price <price>] \
  [--leverage <leverage>] \
  [--tp-pct <pct>] \
  [--sl-pct <pct>] \
  [--time-in-force <tif>]
```

The receiver's supervisor polls the local registry every 5s; the new strategy will get an SSE stream opened automatically within ~5s.

### 7. Confirm to the user

Tell the user:

```
Strategy registered.
  query_id:  <id>
  status:    active
  expires:   <expiresAt from create_query response>

The receiver will pick it up within ~5s and open the SSE stream. When the condition fires, you'll see in-chat alerts (and Telegram push if configured) within seconds. Manual unwind is required after fire (v1 has no auto-close primitive).
```

## When the user wants to cancel

```
elfa-grvt-bot cancel <query_id>
```

Confirm by running `elfa-grvt-bot list --status active` and showing the user the strategy is no longer listed (or showing it with status `cancelled`).

## When the user asks "what's running" or "show me my strategies"

```
elfa-grvt-bot list
```

If they ask about a specific one, query the registry directly (or run `list` and filter visually).

## When the user asks about positions / balance

The local registry does NOT track positions. Fetch live from GRVT:

```python
positions = grvt_client.fetch_positions()
balance = grvt_client.fetch_balance()
open_orders = grvt_client.fetch_open_orders()
```

(The Python SDK wraps these; in your runtime, expose them via the CLI or directly from `cli.py`.)

## Things to never do

- **Never hand-edit EQL.** Builder Chat is the only authority. Re-prompt instead.
- **Never use trade-flavoured Elfa actions** (`market_order`, `limit_order`, `llm` -> trade callback). This bot is notify-only; Elfa would try to execute the trade itself, bypassing the bot. Always frame as `Notify me when: ...`.
- **Never echo secrets back to the user** after they paste them. Write to `.env` via file edit, say "saved", move on.
- **Never bypass the explicit "yes" gate.** Even if the user said "looks good go ahead" five minutes ago, do not POST `create_query` until they say "yes" for THIS specific plan.
- **Never set leverage above what the user requested.** GRVT's max leverage for perps is high; respect what was specified, fall back to account default if not specified.
- **Never assume the registry is in sync with GRVT.** When the user asks about positions, ALWAYS fetch live from GRVT.

## Things to do proactively

- Surface pending alerts at session start, every session.
- Verify symbol exists on GRVT before showing the plan (saves a wasted Builder Chat + validate cycle).
- Compute and show implied notional (size * current_mid) so the user sees the dollar amount at risk.
- Suggest a `max_notional_usd` cap if the user didn't specify (e.g., 20% headroom above the current notional to absorb mid-price drift between authoring and fire).
- When a fire happens (the user will see Telegram or session-start alerts), offer to fetch live positions to confirm fill.
