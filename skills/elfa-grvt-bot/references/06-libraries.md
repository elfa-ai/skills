# 06 - Libraries

Pinned dependencies and what each is trusted for. The implementing agent does NOT need to reimplement anything below; consume each library's documented API and trust it.

## Reference language: Python 3.11+

This spec assumes Python because the reference runtime is in Python and the GRVT SDK is Python-only. Implementations in other languages are possible but must reimplement the GRVT EIP-712 signing path or find an equivalent SDK.

## Hard requirements

| Package | Pin | What it's trusted for |
|---|---|---|
| `grvt-pysdk` | `>=0.2,<1.0` (verified with `0.2.1` on PyPI) | All GRVT interactions: EIP-712 signing of orders, login/cookie management, market data, position/balance queries. **Never hand-roll EIP-712 typed-data signing.** That's how money gets lost. |
| `httpx` | `>=0.27,<1.0` | Async HTTP client for Elfa SSE streams. Streaming via `httpx.AsyncClient.stream("GET", ...)` and `response.aiter_lines()`. Long-lived connections with `timeout=Timeout(connect=10, read=None, write=10, pool=10)`. |
| `requests` | `>=2.31,<3.0` | Sync HTTP client for Elfa REST endpoints (Builder Chat, validate, create, cancel, poll-query) and for the preflight script. Simpler than async for one-shot calls. |
| `python-dotenv` | `>=1.0,<2.0` | Reading `.env` files. Optional: the bootstrap script can also use stdlib parsing if you want zero deps in the installer. |
| `pytest` | `>=7.0,<9.0` (dev) | Test runner for the test vectors in `07-test-vectors.md`. |
| `pytest-asyncio` | `>=0.23,<1.0` (dev) | Async test support. |
| `responses` | `>=0.24,<1.0` (dev) | Mock HTTP for `requests`-based tests. |

## Stdlib-only (do not add a dependency)

| Concern | Stdlib module | Notes |
|---|---|---|
| SQLite | `sqlite3` | Use WAL mode (`PRAGMA journal_mode = WAL;`) on startup. Set busy timeout to 500ms. |
| JSON | `json` | All wire formats are JSON. Never `eval()`. |
| ASCII / UTF-8 | builtin str + bytes | Title and description fields written to Elfa must be ASCII-safe (see SKILL.md conventions). |
| ISO timestamps | `datetime.datetime.utcnow().isoformat()` | All `created_at`, `updated_at`, `received_at`, etc. stored as ISO 8601 strings, UTC, no offset suffix. |
| Decimal arithmetic | `decimal.Decimal` | Use for tick alignment in `04-algorithms.md`. Floats drift on small ticks like 0.0001. |
| CLI argument parsing | `argparse` | For `08-cli.md`'s commands. |
| Async runtime | `asyncio` | The supervisor and strategy loops are async. |
| Process management | `subprocess`, `signal` | Bootstrap script orchestrates `python -m elfa_grvt_bot`. Receiver catches SIGINT/SIGTERM for clean shutdown. |
| Logging | `logging` | Default to INFO level. Format: `%(asctime)s %(levelname)s %(name)s: %(message)s`. Receiver writes to `receiver.log` (line-buffered) or stdout. |

## Canonical GRVT integration patterns: grvt-skills/perpetual-trading

`gravity-technologies/grvt-skills/skills/perpetual-trading` is GRVT's officially published agent skill for trading on the platform. This spec depends on it for every low-level GRVT operation. The implementer should read that skill alongside this spec; the GRVT-specific code (SDK setup, login auto-discovery, `create_order`, `create_trigger_order`, `set_position_config`) is lifted from there. This spec extends those patterns with atomic OTOCO via `bulk_orders` v2; nothing else GRVT-side is original to this spec.

Pin reading order: the perpetual-trading skill's SKILL.md first, then this spec's `02-protocols.md` GRVT section, then `04-algorithms.md`'s order placement section. Implementer copies the helper functions verbatim from the perpetual-trading skill and only writes original code for the bulk_orders submission.

## Verified GRVT SDK surface (2026-05-14)

The live Python package currently exposes `grvt-pysdk==0.2.1`, not a `2.x` line. Before implementing, run a tiny import/signature probe and update this section if it differs:

```python
from pysdk.grvt_ccxt import GrvtCcxt
from pysdk.grvt_ccxt_env import GrvtEnv
from pysdk.grvt_ccxt_utils import get_grvt_order, get_order_payload

# get_grvt_order signature observed in 0.2.1:
# get_grvt_order(sub_account_id, symbol, order_type, side, amount,
#                limit_price, order_duration_secs=300, params={})
# get_order_payload(order, private_key, env, instruments)
```

Use this setup:

```python
params = {
    "api_key": GRVT_TRADING_API_KEY,
    "trading_account_id": GRVT_TRADING_ACCOUNT_ID,
    "private_key": GRVT_TRADING_PRIVATE_KEY,
}
api = GrvtCcxt(env=GrvtEnv.PROD, parameters=params, order_book_ccxt_format=True)
markets = api.fetch_markets(params={"kind": "PERPETUAL"})
instruments = {m.get("id") or m.get("symbol") or m.get("instrument"): m for m in markets}
```

Critical gotchas from a live implementation:

- Pass `params` to `get_grvt_order` as a keyword, never as the 7th positional argument. The 7th positional argument is `order_duration_secs`; passing a dict there fails during signature construction.
- SDK `TimeInForce` enum names are `GOOD_TILL_TIME`, `IMMEDIATE_OR_CANCEL`, and `FILL_OR_KILL`. If the bot CLI accepts `GTC`, `IOC`, `FOK`, map them before calling `get_grvt_order`.
- `get_order_payload(...)` returns a dict whose signed order is under `payload["order"]` in current examples. If the SDK changes shape, fail before submitting any real order.
- Instrument metadata fields such as `tick_size`, `min_size`, and `min_notional` arrive as strings. Parse with `Decimal`.
- Treat `min_size` as both minimum order size and size step unless GRVT exposes a more specific size increment. Order sizes that satisfy min_notional but are not a multiple of this step can be rejected as `Order size too granular`. Example observed on BTC: `min_size="0.001"`; `0.00126 BTC` was rejected, `0.002 BTC` is the next valid size above the 100 USDT min notional.
- Run a signed-payload dry-run during bootstrap: build parent, TP, and SL signed orders locally and print only non-secret structural fields (`instrument`, `size`, `limit_price`, `client_order_id`), then stop before POSTing. This catches helper signature and enum mistakes before an Elfa fire arrives.

## Why we trust grvt-pysdk

EIP-712 signing of GRVT orders is a non-trivial cryptographic protocol:

1. Build a typed-data structure for the `Order` type with all 20+ fields in exact order.
2. Compute the domain separator hash (chain id, contract address, version).
3. Compute the struct hash with `keccak256`.
4. Sign with the EVM private key using ECDSA, producing a `(r, s, v)` triple.
5. Encode the signature into the order body in the exact format GRVT expects.

Any error in any step produces a signature that GRVT rejects, OR (worse) a signature that GRVT accepts but for a different order than you intended. The SDK is the canonical implementation and is updated in lockstep with GRVT's contracts. Use it.

If you absolutely must call `trades.grvt.io` without the SDK (e.g., a non-Python implementation), study the SDK's source and replicate it exactly. Test against a known-good fixture before placing real orders.

## Why we use httpx for SSE specifically

Python's `requests` library does not stream SSE cleanly: it buffers chunks and forces you to parse byte boundaries yourself. `httpx.AsyncClient.stream` with `aiter_lines()` gives you a clean line-oriented async iterator that respects the connection's keep-alive and times out only on the writes/connect (read=None means hold the connection open indefinitely). This matches the SSE protocol's expectations.

Mixing httpx (async, for streams) and requests (sync, for one-shot REST) is intentional: the strategy authoring flow is synchronous (one agent action at a time), and using async there would force the agent's CLI commands to either ship an event loop or spawn one per call. Sync for REST keeps the CLI surface boring.

## Optional dependencies

These are not required but may be useful for specific deployments:

| Package | When to use |
|---|---|
| `uvloop` | Faster asyncio event loop on Linux/macOS. Drop-in replacement: `asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())`. Not required; the bot is not CPU-bound. |
| `structlog` | If you want structured logging out of the box. Otherwise stdlib `logging` is fine. |
| `aiohttp` | An alternative to httpx if you prefer; the SSE-parsing pattern transfers. |

## Pinning policy

This spec uses `>= ; <` ranges, not `==`, because:

- The captured-frames contract locks us against schema drift in production.
- The algorithms in `04-algorithms.md` are language-agnostic; libraries are implementation details.
- Strict pinning forces frequent maintenance PRs without catching real regressions.

For production deployments, pin in `pyproject.toml` with `~=` (compatible-release operator) and let `pip-compile` or `uv` generate a `requirements.txt` with strict pins. The implementing agent should set this up in step 1 of the implementation order.

## Forbidden libraries

These should NOT appear in any version of the runtime:

| Package | Why not |
|---|---|
| `eth-account`, `eth-keys`, `web3` (direct use) | If you need EVM signing, use grvt-pysdk. Don't roll your own typed-data signing. |
| `pycryptodome` (for signing-related primitives) | Same. The SDK has what you need. |
| `flask`, `fastapi`, `aiohttp.web`, `starlette`, any HTTP server | The bot has no inbound HTTP surface. If you find yourself reaching for these, you are reimplementing the old webhook architecture. Don't. Triggers arrive over outbound SSE; see `01-architecture.md` and `02-protocols.md`. |
| `cloudflared`, `ngrok`, `localtunnel`, or any tunneling tool | The bot does not need a public URL. If you think you need one, you have misread the architecture. Re-read `01-architecture.md`. |
| `celery`, `rq`, any task queue | The supervisor + asyncio is the task model. Don't add infrastructure. |
| `redis`, `postgres`, any other DB | SQLite is the registry. One file, one process, one source of truth. |

If a future requirement seems to need any of these, write it down and discuss with maintainers before adding the dependency.

## Example pyproject.toml skeleton

```toml
[project]
name = "elfa-grvt-bot"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = [
    "grvt-pysdk>=0.2,<1.0",
    "httpx>=0.27,<1.0",
    "requests>=2.31,<3.0",
    "python-dotenv>=1.0,<2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0,<9.0",
    "pytest-asyncio>=0.23,<1.0",
    "responses>=0.24,<1.0",
]

[project.scripts]
elfa-grvt-bot = "elfa_grvt_bot.cli:main"

[build-system]
requires = ["setuptools>=64", "wheel"]
build-backend = "setuptools.build_meta"

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

The `elfa-grvt-bot` console script is the CLI entry point described in `08-cli.md`.
