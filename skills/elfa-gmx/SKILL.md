---
name: elfa-gmx
description: >
  Trade GMX perpetuals through Elfa Auto — condition-driven strategies (price, TA, cron,
  LLM, social signals) that execute market/limit orders with TP/SL on GMX, using only the
  Elfa API (no GMX API). Covers setup (API key, Auto enablement, HMAC, GMX connection),
  GMX-specific action parameters, and the GMX Masters competition rules and
  disqualification guardrails. Trigger when the user wants to trade or build a bot on GMX
  via Elfa, automate GMX perp entries/exits, or join or compete in GMX Masters / the
  Elfa x GMX competition.
---

# elfa-gmx — trade GMX through Elfa Auto

Build automated GMX perp trading on top of [Elfa Auto](https://docs.elfa.ai/auto/overview):
describe a condition (price level, indicator, schedule, LLM predicate, social signal), and
Auto executes a market or limit order on GMX when it fires. Everything goes through the
Elfa API — **no GMX API, no direct contract calls**. Elfa handles order routing and
execution on GMX.

Works standalone for any GMX automation, and doubles as the playbook for the **GMX
Masters** competition (last section) — same mechanics, plus rules.

## Companion skill: `elfa-ai`

All API mechanics live in the **`elfa-ai` skill** — endpoints, Builder Chat, the EQL query
model, condition sources, HMAC signing, lifecycle order, error handling. Use it for every
API call. If not installed, fetch it from <https://github.com/elfa-ai/skills> or fall back
to <https://docs.elfa.ai>. This skill only adds the GMX context on top.

## Setup checklist

Steps marked **(user, in browser)** cannot be done by the agent — share the link and wait.

1. **Elfa API key** (user, in browser): create one at <https://dev.elfa.ai/>. Set as
   `ELFA_API_KEY` (env var, never pasted into chat).
2. **Enable Auto + HMAC secret** (user, in browser): in the same portal, select the
   API key → **Auto** tab → **Sign in to Enable Auto** (Privy login) → copy the HMAC
   secret (shown once). Set as `ELFA_HMAC_SECRET`. Trade-action mutations require HMAC.
3. **Connect GMX** (user, in browser): from the portal's **Exchange Connections**, open the
   Elfa app with the same identity, complete GMX onboarding (deposit funds), then click
   **Verify connection** back in the portal.
4. **Verify trade-readiness** (agent): `GET /v2/auto/exchanges` must show `gmx` active.
   Without it, trade queries fail at execution time with `AGENT_WALLET_REQUIRED`.

## Building strategies

Follow the `elfa-ai` skill's Auto flow (Builder Chat → validate → create → monitor).
GMX-specific parts:

- Trade actions must target GMX: `"exchange": "gmx"` in `market_order` / `limit_order`
  params. Confirm symbols with `GET /v2/auto/validate-symbol/gmx/:symbol`.
- GMX rejects `reduceOnly` and `marginType` — omit both. `leverage`, `tp`, `sl` are
  supported. No fixed minimum order notional.
- `price` / `ta` conditions can source data from either venue (`gmx` or `hyperliquid`) —
  the condition's data venue is independent of where the order executes.
- Monitor fires via `GET /v2/auto/executions` or the per-query SSE stream. Elfa does not
  proxy GMX account state; read positions/balance on-chain or in the Elfa app.
- For direct, non-conditional execution (place/close/TP-SL now), the `/v2/trade/*`
  endpoints support GMX too — see the `elfa-ai` skill's Trade section.

Strategy shape (how many queries, which conditions, sizing) is the agent's call, shaped by
what the user wants.

## GMX Masters (competition)

4-week competition by Elfa and GMX, 15 June 16:00 UTC – 12 July 15:59 UTC 2026 (shown in the
docs as 16 June – 12 July, UTC+8 — same instants), 15,000 USDC across four tracks: Auto
Trading ($10,000, ranked by percentage return on capital), Alpha Pool ($3,000, best
strategies shared on X), Subscriber Bonus ($1,000), Partner Bonus ($1,000). Strategies run on
Elfa Auto executing on GMX — exactly the flow above. Leaderboard is public, updates every 2
hours. Prizes pay out at the end of the competition, **except** the Subscriber Draw, which
pays 1 month after the participant's payment. Rules source of truth:
<https://docs.elfa.ai/gmx-trading-competition/faq> — the live docs win over this skill.

**Concurrent Volume Competition (separate pop-up, no registration).** Running alongside GMX
Masters is a $1,500 volume competition — you're **auto-enrolled** on any GMX trade through
Elfa (no GMX Masters signup required). Prizes: 1st $1,000, 2nd 3-month Max membership, 3rd
1-month Max membership, plus a $500 raffle for any trader with >$500k GMX volume via Elfa.
All trades must execute on GMX through Elfa. Details:
<https://docs.elfa.ai/volume-competition/faq>.

### The one rule that shapes everything

**Every competition trade must execute on GMX through Elfa Auto** — not directly against
GMX contracts, not through the GMX UI or any other frontend or bot.

Why: Elfa attributes competition activity on-chain via GMX's `uiFeeReceiver` field. Orders
placed through Elfa Auto are stamped with Elfa's receiver address automatically — its
absence is how out-of-band trading is detected, flagging the account for
disqualification. Never set `uiFeeReceiver` yourself — route everything through Elfa
Auto.

### Extra setup for competitors

1. **Register** (user, in browser): <https://go.elfa.ai/gmx-masters> — required for any
   prize track. Same account identity as the API key / Auto enablement.
2. **Account state at join** (user, checked when joining): minimum balance 500 USDC on
   Arbitrum; no open positions or pending orders; no GMX liquidity (GM or GLV) held.
3. **At least 1 Auto plan** created during the competition window — verified on Elfa's
   backend, required for any prize track.

### Disqualification rules (bake into everything you build)

- **No trading GMX outside Elfa** (e.g. directly on the GMX app) during the competition
  window.
- **No suspicious capital movement** — deposits, withdrawals, or sub-account activity
  after joining may be reviewed and lead to disqualification. Never suggest topping up
  mid-competition — size strategies within the existing balance.
- **No wash trading, collusion, or artificial inflation of returns.**
