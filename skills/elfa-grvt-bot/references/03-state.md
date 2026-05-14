# 03 - State

The bot persists three things to a single SQLite database at `REGISTRY_DB_PATH` (default `./registry.db`). The schema below is the binding contract: column names, types, and constraints are referenced by name throughout the algorithms in `04-algorithms.md`. Do not rename or restructure without updating the algorithms in lockstep.

## Schema (DDL)

```sql
-- Active and historical strategies. Inserted by the authoring flow,
-- transitioned by the receiver as fires arrive or terminal state is
-- observed remotely.
CREATE TABLE IF NOT EXISTS strategies (
    query_id              TEXT PRIMARY KEY,        -- Elfa query.id (UUID)
    title                 TEXT NOT NULL,
    description           TEXT,
    eql_json              TEXT NOT NULL,           -- the full inner EQL as JSON

    -- GRVT order spec
    symbol                TEXT NOT NULL,           -- e.g. BNB_USDT_Perp
    side                  TEXT NOT NULL CHECK (side IN ('buy','sell')),
    amount                REAL NOT NULL CHECK (amount > 0),
    order_type            TEXT NOT NULL CHECK (order_type IN ('market','limit')),
    price                 REAL,                    -- only for order_type='limit'
    leverage              INTEGER,                 -- optional; NULL = use account default
    time_in_force         TEXT DEFAULT 'GTC',
    reduce_only           INTEGER NOT NULL DEFAULT 0,  -- boolean

    -- Guardrails
    max_notional_usd      REAL NOT NULL CHECK (max_notional_usd > 0),

    -- Optional TP/SL (percentages relative to fill, computed at fire time)
    tp_pct                REAL,                    -- e.g. 1.5 -> 1.5% from fill
    sl_pct                REAL,                    -- e.g. 1.0 -> 1.0% from fill

    -- Environment
    env                   TEXT NOT NULL DEFAULT 'prod' CHECK (env = 'prod'),

    -- Lifecycle
    status                TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active','fired','expired','cancelled','failed')),
    created_at            TEXT NOT NULL,           -- ISO 8601, UTC
    updated_at            TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_strategies_status ON strategies(status);

-- Fire records: one row per executionId observed (via SSE or poll-query).
-- Primary key is the dedupe boundary. Inserting a row with an existing
-- event_id is a no-op (used to dedupe).
CREATE TABLE IF NOT EXISTS fires (
    event_id              TEXT PRIMARY KEY,        -- SSE data.executionId (UUID)
    query_id              TEXT NOT NULL,
    raw_payload           TEXT NOT NULL,           -- full SSE data: JSON, for debugging

    -- Outcome of the fire processing
    outcome               TEXT NOT NULL
        CHECK (outcome IN ('placed','rejected_guardrail','grvt_error',
                           'unknown_strategy','duplicate','unknown')),
    error                 TEXT,                    -- error string if outcome != placed

    -- GRVT order ids if placed
    parent_order_id       TEXT,
    tp_order_id           TEXT,
    sl_order_id           TEXT,

    -- Reference prices at fire time (for audit)
    reference_price       REAL,
    tp_price              REAL,
    sl_price              REAL,

    received_at           TEXT NOT NULL,           -- when the bot received the SSE frame
    placed_at             TEXT                     -- when GRVT accepted the order (if placed)
);

CREATE INDEX IF NOT EXISTS idx_fires_query_id ON fires(query_id);

-- User-facing alerts. Surfaced via in-chat pull and optionally Telegram push.
-- Acked alerts stay queryable for audit but are filtered from new sessions.
CREATE TABLE IF NOT EXISTS alerts (
    id                    INTEGER PRIMARY KEY AUTOINCREMENT,
    severity              TEXT NOT NULL CHECK (severity IN ('info','warning','error')),
    category              TEXT NOT NULL,           -- see categories list in 01-architecture.md
    message               TEXT NOT NULL,
    query_id              TEXT,                    -- nullable: not all alerts pertain to a strategy
    fire_event_id         TEXT,                    -- nullable: only set when alert pertains to a fire
    details_json          TEXT,                    -- optional structured detail blob
    created_at            TEXT NOT NULL,
    acked_at              TEXT                     -- NULL = unacked
);

CREATE INDEX IF NOT EXISTS idx_alerts_unacked ON alerts(acked_at) WHERE acked_at IS NULL;
```

Schema migrations are out of scope for v1: the DDL above must be created idempotently on every receiver startup and CLI invocation. If columns change in a future version, design a migration strategy then.

## Status state machine

`strategies.status` transitions:

```
active --SSE fire processed (placed or rejected)--> fired
active --poll-query reports remote 'triggered'---> fired
active --poll-query reports remote 'expired'-----> expired
active --user 'cancel' CLI / cancel_query--------> cancelled
active --poll-query reports remote 'cancelled'---> cancelled
active --poll-query reports remote 'failed'------> failed
active --poll-query reports remote 'recurring'---> failed (bot does not support recurring)
active --SSE 404 / EQL validation broken---------> failed
```

All terminal statuses are absorbing: no transition out. The bot never re-arms a strategy. To rerun the same intent, the user authors a new strategy.

The `updated_at` column must be touched on every status transition. `created_at` is set on insert and never changes.

## Idempotency invariant

**Every (query_id, executionId) pair maps to exactly one `fires` row.** When the SSE parser yields a trigger event, the receiver attempts to insert a fires row with `event_id = executionId, outcome = 'unknown'` BEFORE doing any order placement. The insert is wrapped:

```python
try:
    INSERT INTO fires (event_id, query_id, outcome, received_at, raw_payload)
    VALUES (?, ?, 'unknown', ?, ?);
except sqlite3.IntegrityError:  # PK collision
    # duplicate event; do nothing
    return
```

If the insert succeeds, the bot proceeds to place the order and then UPDATEs the row with the outcome. If the insert fails with a PK collision, that executionId has already been processed (perhaps the SSE redelivered, perhaps a poll-query reconcile noticed it first); skip silently. This makes order placement at-most-once even across SSE reconnects and crash recoveries.

Poll-query reconciliation also goes through the same dedupe path: when poll-query reports `executions[i]`, the receiver checks `SELECT 1 FROM fires WHERE event_id = ?`. If present, no action. If absent and remote status is terminal-triggered, the bot emits a `manual_intervention_required` alert (and inserts a fires row with `outcome = 'unknown'` to suppress future duplicates).

## Invariants the schema doesn't enforce (must be enforced in code)

1. **A strategy in `fired` status has at least one fires row with `outcome IN ('placed', 'rejected_guardrail', 'grvt_error', 'unknown')` for its query_id**, OR was synced via the manual-intervention path.
2. **A `fires` row with `outcome = 'placed'` has non-null `parent_order_id`** (and `tp_order_id`, `sl_order_id` if the strategy had `tp_pct`/`sl_pct`).
3. **`alerts.fire_event_id`, when set, references a row in `fires.event_id`**. The schema does not enforce this FK because alerts can be created from poll-query reconciliation before the fires row exists (briefly); the application orders these correctly but the FK is intentionally loose.
4. **`raw_payload` is valid JSON** parseable to a dict. The parser only yields events where this is true, so any code reading `raw_payload` may assume it without re-validating.

## Concurrency

Authoring (the agent's CLI invocations) and the receiver process both write to the same database. SQLite handles this with file-level locking; the bot uses default WAL mode for better concurrent reads. Both processes must:

- Open the database with a short busy timeout (~500ms) and retry on `SQLITE_BUSY`.
- Wrap multi-statement transitions in transactions (e.g., updating `strategies.status` and inserting an `alerts` row).
- Never hold a write lock across an external API call (do the network call first, then write the result in a single fast transaction).

## Sensitive data

The registry does NOT store API keys, private keys, or any secret. `.env` is the only place secrets live. The schema deliberately has no column for credentials. Anyone who reads the registry sees strategy specs and audit history; that's it.

## A note on schema validation

The bot's startup sequence runs the DDL idempotently (`CREATE TABLE IF NOT EXISTS ...`). If a user manually edited the database and a column is missing, behavior is undefined. Don't do that.

If the user moves `registry.db` between hosts, the schema goes with it; no migration is needed for v1.
