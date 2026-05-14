# 08 - CLI

User-facing commands. Implement these as subcommands of a single `elfa-grvt-bot` entry point. Where possible the agent runs these on the user's behalf during the install walkthrough.

The CLI is designed for **local install**: the user runs these on their own machine after cloning / installing the runtime. There is no remote-control surface, no daemon API, no inbound socket. Everything is invoked from the same shell session, against the same `registry.db` file. If the user later wants to run the receiver under systemd / Fly.io / etc., the same commands work unchanged (in particular `run --daemon` for background execution), but coordinating that across hosts is out of scope for this skill.

## Entry point

```
elfa-grvt-bot <subcommand> [options]
```

Defined as a `[project.scripts]` entry in `pyproject.toml` so it's available after `pip install -e .`. The implementation is in `elfa_grvt_bot/cli.py`'s `main()` function.

## Subcommands

### `init [target_dir]`

Scaffold a working directory. Creates `.env` from `.env.example`, runs the registry DDL idempotently, drops in the `AGENTS.md` template.

```
elfa-grvt-bot init ~/elfa_grvt_bot
```

If `target_dir` is omitted, defaults to `~/elfa_grvt_bot`.

Does NOT install dependencies, run preflight, or start the receiver. Those are separate explicit commands so the user can see what's happening at each step.

Output:

```
[ok] target: /home/user/elfa_grvt_bot
[ok] wrote .env from .env.example (fill in credentials before running preflight)
[ok] wrote AGENTS.md (chat-flow instructions for your preferred agent)
[ok] created registry.db with schema
[!!] next step: fill in .env, then run `elfa-grvt-bot preflight`
```

### `preflight`

Run the three probes from `02-protocols.md` and `05-failure-modes.md`. Cheap, ~1 second total. Exits 0 if all pass, non-zero with a clear message otherwise.

```
cd ~/elfa_grvt_bot
elfa-grvt-bot preflight     # auto-loads .env from cwd
```

Output on success:

```
preflight: probing external dependencies

[ok] elfa: API key accepted
[ok] grvt: API key accepted, cookie issued
[ok] telegram: bot @your_bot reachable

preflight: all probes passed
```

Output on failure (example: geo-block):

```
preflight: probing external dependencies

[ok] elfa: API key accepted
[!!] grvt: GEO-BLOCK: Access from this location is not allowed.  (deploy from an allowed region or VPN)
[ok] telegram: skipped (not configured, alerts will be in-chat only)

preflight: FAIL
  fix the failed probes above before starting the receiver.
```

### `run`

Start the receiver in the foreground (or detach with `--daemon`). Long-running. Catches SIGINT/SIGTERM for clean shutdown.

```
elfa-grvt-bot run               # foreground, log to stdout
elfa-grvt-bot run --daemon      # background, writes PID file and log file
```

Implementation: foreground mode is just the asyncio `run(supervisor(...))` call. Daemon mode forks (or uses `subprocess.Popen` with detachment flags), writes `.receiver.pid` and `receiver.log`, and exits the parent so the user gets their prompt back. The daemon path is what `bootstrap` uses.

### `bootstrap [target_dir]`

Convenience: `init` + dep install + preflight + `run --daemon` in one shot. Idempotent.

```
elfa-grvt-bot bootstrap ~/elfa_grvt_bot
```

Phases (similar to the previous `scripts/bootstrap.py`):

1. `init` (scaffold)
2. Create venv + install deps (`pip install -e ".[dev]"`)
3. Run tests (skippable with `--skip-tests`)
4. Validate `.env` has every required var; if not, print missing vars and exit (user fills them in, re-runs bootstrap)
5. `preflight` (fail loud if any probe fails)
6. `doctor order-builder` (build signed parent/TP/SL payloads locally, do not POST)
7. `run --daemon` (start receiver)
8. Print summary with PID, log path, teardown command

This is the command the install walkthrough in `SKILL.md` should call.

### `teardown`

Stop a running daemon receiver.

```
elfa-grvt-bot teardown
```

Reads `.receiver.pid`, sends SIGTERM, waits up to 5 seconds, then SIGKILL if it didn't exit. Removes the PID file on success.

### `list`

Print all strategies in the registry. Shows query_id, symbol, side, amount, env, status, title.

```
elfa-grvt-bot list                  # all strategies
elfa-grvt-bot list --status active  # filter
```

### `add ...`

Add a strategy row to the registry. Called by the authoring flow after `POST /v2/auto/queries` returns. The agent calls this; the user rarely does directly.

```
elfa-grvt-bot add \
  --query-id <uuid> \
  --title "..." \
  --description "..." \
  --eql-json '<json>' \
  --symbol BNB_USDT_Perp \
  --side buy \
  --amount 0.02 \
  --order-type market \
  --max-notional-usd 30 \
  [--price <float>] \
  [--leverage <int>] \
  [--tp-pct <float>] \
  [--sl-pct <float>] \
  [--time-in-force GTC|IOC|FOK] \
  [--reduce-only]
```

Validates the inputs against the schema constraints in `03-state.md`. Errors out with a clear message if any constraint is violated (negative amount, non-prod env, etc.).

### `cancel <query_id>`

Cancel a strategy, both locally and on Elfa.

```
elfa-grvt-bot cancel f12c99e3-c9dc-4aa0-85bd-a152a92f5bd3
```

Steps:
1. `POST /v2/auto/queries/<id>/cancel` (accepts 200 or 409-already-terminal).
2. Update local `strategies.status` to `cancelled`.
3. The receiver's supervisor will reap the per-strategy task on its next reconcile (~5s).

### `alerts [--pending|--all] [--json]`

Print alerts. Default shows only unacked.

```
elfa-grvt-bot alerts --pending      # default
elfa-grvt-bot alerts --all          # include acked
elfa-grvt-bot alerts --pending --json  # for agent consumption
```

Format (text):

```
* #3 [warning/guardrail_rejected] fire rejected locally: notional $500.00 exceeds max_notional_usd $100.00
     query_id=359cd110-9cb6-4be6-ac16-99401d13d998
* #2 [info/order_placed] BUY 0.02 BNB_USDT_Perp (market) placed on GRVT. order=ord_abc123def456
     query_id=0591b42b-056c-47bd-b931-ef0da8fa2dff
* #1 [info/trigger_received] Elfa trigger fired: Smoke test #2: BNB long after 5min cron
     query_id=0591b42b-056c-47bd-b931-ef0da8fa2dff
```

### `ack <id|all>`

Mark alert(s) as acknowledged. They stay in the table for audit but are excluded from `--pending`.

```
elfa-grvt-bot ack 3
elfa-grvt-bot ack all
```

### `smoke-test`

Run the documented end-to-end smoke test (see SKILL.md section). Creates a tiny strategy that fires fast, watches for the order, prints the result, optionally unwinds (`--no-unwind` to leave the position open for review).

```
elfa-grvt-bot smoke-test --symbol BTC_USDT_Perp --amount auto --max-notional-usd auto
```

Walks through:
1. Create cron.once 1m or 5m strategy.
2. Watch receiver log + alerts table for fire.
3. Confirm order_placed alert appears.
4. Optionally market-sell to unwind.

This is the "did everything work" final check the bootstrap walkthrough should offer.

If `--amount auto`, choose the smallest valid amount using GRVT metadata:

1. Fetch `min_size`, `min_notional`, `tick_size`.
2. Fetch current mid.
3. Compute `ceil(min_notional / mid / min_size) * min_size` with `Decimal`.
4. Set `max_notional_usd` to computed notional plus small headroom, then ask for explicit user confirmation before creating the Elfa query.

Never pick a smoke amount by notional alone. BTC `0.00126` can be above 100 USDT but invalid if `min_size` is `0.001`; use `0.002`.

### `doctor order-builder`

Build signed GRVT order payloads without submitting them. This catches SDK signature, enum, TIF, and payload-shape mistakes before a real Elfa fire.

```
elfa-grvt-bot doctor order-builder --symbol BTC_USDT_Perp --side buy --amount 0.002 --tp-pct 1 --sl-pct 1
```

Output should include only non-secret structural checks:

```
[ok] grvt sdk imports
[ok] market metadata: min_size=0.001 min_notional=100 tick_size=0.1
[ok] amount multiple of min_size
[ok] notional above min_notional
[ok] parent signed payload built
[ok] tp/sl signed payloads built
```

Do not print private keys, cookies, signatures, or full payloads.

## Env-loading convention

All commands above must auto-source `.env` from the current working directory at startup. The reference Python:

```python
def load_env():
    env_path = Path.cwd() / ".env"
    if env_path.exists():
        from dotenv import load_dotenv
        load_dotenv(env_path)
    # If the user already has the vars exported, that wins; load_dotenv
    # does not override existing env vars.
```

This means the user does NOT have to remember `set -a && source .env && set +a` before each command. Implementations that require explicit env sourcing are buggy.

## `.env.example`

Ship this as part of `init`:

```ini
# Elfa
ELFA_API_KEY=

# GRVT (this project is prod-only; do not change GRVT_ENV)
# Use the "Trading API Key" type from GRVT's Settings -> API Keys.
# GRVT_TRADING_ACCOUNT_ID is auto-discovered by `elfa-grvt-bot preflight`
# from the login response (sub_account_id field). Do NOT paste it manually.
GRVT_TRADING_API_KEY=
GRVT_TRADING_PRIVATE_KEY=
GRVT_TRADING_ACCOUNT_ID=
GRVT_ENV=prod

# Telegram (OPTIONAL real-time push). Leave both blank to disable
# Telegram entirely; in-chat alerts via the registry still work.
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=

# Receiver
REGISTRY_DB_PATH=./registry.db
```

`init` copies this to `.env` only if `.env` does not already exist (do not clobber an existing file). `preflight` populates `GRVT_TRADING_ACCOUNT_ID` automatically from the login response if it is blank.

## Logging convention

`elfa-grvt-bot run` logs to stdout by default. `elfa-grvt-bot run --daemon` redirects stdout to `receiver.log` in cwd. Both use the same format:

```
2026-05-13 14:30:59,970 INFO elfa_grvt_bot.receiver: supervisor started (poll_interval=5.0s)
2026-05-13 14:30:59,970 INFO elfa_grvt_bot.receiver: spawning SSE task for f12c99e3-c9dc-4aa0-85bd-a152a92f5bd3
```

Subcommands like `list`, `alerts`, `add` write to stdout in human-readable text. Adding `--json` to any of those switches to machine-readable JSON for agent consumption.
