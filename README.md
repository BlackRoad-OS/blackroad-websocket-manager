# blackroad-websocket-manager

> **Production-grade WebSocket connection tracker for BlackRoad OS agents.**  
> Real-time connection pooling, heartbeat monitoring, message history, and broadcast — backed by SQLite. Designed for scale, ready for npm wrappers and Stripe webhook delivery pipelines.

[![CI](https://github.com/BlackRoad-OS/blackroad-websocket-manager/actions/workflows/ci.yml/badge.svg)](https://github.com/BlackRoad-OS/blackroad-websocket-manager/actions/workflows/ci.yml)
[![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/downloads/)
[![License: Proprietary](https://img.shields.io/badge/license-Proprietary-red.svg)](./LICENSE)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Features](#2-features)
3. [Installation](#3-installation)
4. [Quick Start](#4-quick-start)
5. [CLI Reference](#5-cli-reference)
   - 5.1 [connect](#51-connect)
   - 5.2 [disconnect](#52-disconnect)
   - 5.3 [list](#53-list)
   - 5.4 [broadcast](#54-broadcast)
   - 5.5 [send](#55-send)
   - 5.6 [heartbeat](#56-heartbeat)
   - 5.7 [heartbeat-check](#57-heartbeat-check)
   - 5.8 [history](#58-history)
   - 5.9 [stats](#59-stats)
6. [Python API Reference](#6-python-api-reference)
   - 6.1 [get_db](#61-get_db)
   - 6.2 [ConnectionPool](#62-connectionpool)
   - 6.3 [add_connection](#63-add_connection)
   - 6.4 [remove_connection](#64-remove_connection)
   - 6.5 [broadcast](#65-broadcast)
   - 6.6 [send_message](#66-send_message)
   - 6.7 [heartbeat_check](#67-heartbeat_check)
   - 6.8 [get_message_history](#68-get_message_history)
   - 6.9 [connection_stats](#69-connection_stats)
7. [Database Schema](#7-database-schema)
8. [npm / Node.js Integration](#8-npm--nodejs-integration)
9. [Stripe Webhook Integration](#9-stripe-webhook-integration)
10. [Production Deployment](#10-production-deployment)
11. [Testing](#11-testing)
12. [Contributing](#12-contributing)
13. [License](#13-license)

---

## 1. Overview

`blackroad-websocket-manager` is the core connection-tracking layer for BlackRoad OS. It provides a persistent, queryable record of every WebSocket session across all agents — along with broadcast, direct-message, heartbeat, and history facilities.

It ships as a standalone Python module with a CLI, a Python API, and a well-defined SQLite schema that any consumer (Node.js, Go, Rust, or a Stripe webhook handler) can query directly.

---

## 2. Features

| Feature | Details |
|---|---|
| **Connection tracking** | `Connection` dataclass with agent, metadata, heartbeat, and message-count fields |
| **In-memory pool** | `ConnectionPool` keyed on `ws_id`, rehydrated from SQLite on start |
| **Persistent storage** | SQLite with `connections`, `messages`, and `heartbeat_log` tables |
| **Indexed queries** | Indexes on `agent`, `status`, `recipient_id`, and `sent_at` for fast lookups |
| **Broadcast** | Fan-out to all connections or a filtered subset |
| **Direct message** | Single-target delivery with full delivery tracking |
| **Heartbeat monitoring** | Per-connection timestamps; automatic stale-connection removal |
| **Message history** | Full inbox/outbox per connection, paginated |
| **Stats endpoint** | Live counts of active connections, total messages, and per-agent breakdowns |
| **CLI** | Full-featured command-line interface for ops and automation |

---

## 3. Installation

**Requirements:** Python 3.11+, `pytest` (for tests only).

```bash
# Clone the repository
git clone https://github.com/BlackRoad-OS/blackroad-websocket-manager.git
cd blackroad-websocket-manager

# No external runtime dependencies — stdlib only.
# Install test dependencies:
pip install pytest
```

The default database is created automatically at `~/.blackroad/websocket-manager.db` on first use.  
Override with `--db /path/to/custom.db` on any command.

---

## 4. Quick Start

```python
from src.main_module import get_db, ConnectionPool, add_connection, broadcast

# Open (or create) the database
db = get_db()
pool = ConnectionPool(db)

# Register a connection
conn = add_connection(pool, ws_id="ws-001", agent="octavia", metadata={"region": "us-west"})
print(conn.ws_id, conn.agent)  # ws-001 octavia

# Broadcast an event to all active connections
delivered = broadcast(pool, db, {"event": "deploy", "version": "2.0.0"})
print(f"Delivered to {len(delivered)} connection(s)")
```

---

## 5. CLI Reference

All commands accept a global `--db` flag to point at a custom SQLite file.

```
python src/main_module.py [--db PATH] <command> [options]
```

### 5.1 `connect`

Register a new WebSocket connection.

```bash
python src/main_module.py connect <agent> [--ws-id WS_ID] [--metadata JSON]
```

| Argument | Required | Description |
|---|---|---|
| `agent` | Yes | Agent name (e.g. `octavia`) |
| `--ws-id` | No | WebSocket ID; auto-generated UUID if omitted |
| `--metadata` | No | JSON string of arbitrary metadata (default: `{}`) |

```bash
python src/main_module.py connect octavia --ws-id ws-001 --metadata '{"ip":"10.0.0.1"}'
# Connected: ws-001 (agent=octavia)
```

### 5.2 `disconnect`

Mark a connection as disconnected and remove it from the active pool.

```bash
python src/main_module.py disconnect <ws_id>
```

### 5.3 `list`

List all currently active connections.

```bash
python src/main_module.py list
```

### 5.4 `broadcast`

Broadcast a message to all active connections, or to a filtered subset by agent.

```bash
python src/main_module.py broadcast <message> [--agent AGENT] [--type MSG_TYPE]
```

```bash
# Broadcast to all
python src/main_module.py broadcast '{"event":"deploy","version":"1.2.3"}'

# Broadcast to a specific agent only
python src/main_module.py broadcast "hello" --agent octavia
```

### 5.5 `send`

Send a direct message to a single connection by `ws_id`.

```bash
python src/main_module.py send <ws_id> <message>
```

```bash
python src/main_module.py send ws-001 '{"ping":true}'
```

### 5.6 `heartbeat`

Record a heartbeat for a connection, optionally with a latency measurement.

```bash
python src/main_module.py heartbeat <ws_id> [--latency MS]
```

```bash
python src/main_module.py heartbeat ws-001 --latency 5
```

### 5.7 `heartbeat-check`

Sweep all active connections and remove any that have not sent a heartbeat within the timeout window.

```bash
python src/main_module.py heartbeat-check [--timeout SECONDS]
```

```bash
python src/main_module.py heartbeat-check --timeout 30
# Active: 4  Timed out: 1
#   Removed: ws-stale-007
```

### 5.8 `history`

View message history, optionally filtered to a single connection.

```bash
python src/main_module.py history [--ws-id WS_ID] [--limit N]
```

```bash
python src/main_module.py history --ws-id ws-001 --limit 20
```

### 5.9 `stats`

Print a JSON summary of connection and message counts.

```bash
python src/main_module.py stats
# {
#   "active_connections": 3,
#   "total_ever_connected": 17,
#   "total_messages": 204,
#   "agents": { "octavia": 2, "aria": 1 }
# }
```

---

## 6. Python API Reference

### 6.1 `get_db`

```python
get_db(db_path: Path = DB_PATH) -> sqlite3.Connection
```

Open (or create) the SQLite database at `db_path`, initialize the schema, and return a `sqlite3.Connection` with `row_factory = sqlite3.Row`.

---

### 6.2 `ConnectionPool`

```python
class ConnectionPool:
    def __init__(self, conn: sqlite3.Connection): ...
    def add(self, connection: Connection) -> Connection: ...
    def remove(self, ws_id: str) -> bool: ...
    def get(self, ws_id: str) -> Optional[Connection]: ...
    def get_all(self) -> List[Connection]: ...
    def count(self) -> int: ...
    def update_heartbeat(self, ws_id: str, latency_ms: Optional[int] = None) -> bool: ...
    def increment_message_count(self, ws_id: str) -> None: ...
```

In-memory pool backed by SQLite. On instantiation, rehydrates all `status='active'` rows from the database.

---

### 6.3 `add_connection`

```python
add_connection(
    pool: ConnectionPool,
    ws_id: str,
    agent: str,
    metadata: Optional[Dict] = None,
) -> Connection
```

Create and persist a new `Connection`. Upserts on `ws_id`.

---

### 6.4 `remove_connection`

```python
remove_connection(pool: ConnectionPool, ws_id: str) -> bool
```

Mark a connection as `disconnected` in the database and evict it from the in-memory pool. Returns `False` if `ws_id` is not found.

---

### 6.5 `broadcast`

```python
broadcast(
    pool: ConnectionPool,
    db_conn: sqlite3.Connection,
    message: Any,
    filter_fn: Optional[Callable[[Connection], bool]] = None,
    msg_type: str = "broadcast",
    sender_id: Optional[str] = None,
) -> List[str]
```

Fan-out `message` to all active connections (or a filtered subset). Persists each delivery to the `messages` table and increments per-connection `message_count`. Returns the list of `ws_id`s that received the message.

---

### 6.6 `send_message`

```python
send_message(
    pool: ConnectionPool,
    db_conn: sqlite3.Connection,
    ws_id: str,
    message: Any,
    msg_type: str = "data",
    sender_id: Optional[str] = None,
) -> Optional[Message]
```

Deliver a message to a single connection. Returns the persisted `Message` object, or `None` if the connection is not found.

---

### 6.7 `heartbeat_check`

```python
heartbeat_check(
    pool: ConnectionPool,
    db_conn: sqlite3.Connection,
    timeout: int = 30,
) -> Dict[str, List[str]]
```

Sweep all active connections. Any connection whose `last_heartbeat` is older than `timeout` seconds is removed from the pool and marked `disconnected`. Returns `{"active": [...], "timed_out": [...]}`.

---

### 6.8 `get_message_history`

```python
get_message_history(
    db_conn: sqlite3.Connection,
    ws_id: Optional[str] = None,
    limit: int = 50,
) -> List[dict]
```

Return up to `limit` messages ordered by `sent_at DESC`. Optionally filter to messages sent to or from `ws_id`.

---

### 6.9 `connection_stats`

```python
connection_stats(pool: ConnectionPool, db_conn: sqlite3.Connection) -> dict
```

Return a summary dict:

```json
{
  "active_connections": 3,
  "total_ever_connected": 17,
  "total_messages": 204,
  "agents": { "octavia": 2, "aria": 1 }
}
```

---

## 7. Database Schema

The database is a single SQLite file (default: `~/.blackroad/websocket-manager.db`).

### `connections`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `ws_id` | TEXT UNIQUE | WebSocket session identifier |
| `agent` | TEXT | Owning agent name |
| `metadata` | TEXT | JSON blob |
| `connected_at` | TEXT | ISO-8601 timestamp |
| `last_heartbeat` | TEXT | ISO-8601 timestamp |
| `status` | TEXT | `active` \| `disconnected` |
| `message_count` | INTEGER | Cumulative messages delivered |
| `disconnected_at` | TEXT | ISO-8601 timestamp or NULL |

**Indexes:** `idx_conn_agent (agent)`, `idx_conn_status (status)`

### `messages`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `msg_id` | TEXT UNIQUE | UUID |
| `msg_type` | TEXT | `data` \| `broadcast` \| `ping` \| … |
| `sender_id` | TEXT | Nullable `ws_id` |
| `recipient_id` | TEXT | Nullable `ws_id` |
| `content` | TEXT | JSON or plain string |
| `sent_at` | TEXT | ISO-8601 timestamp |
| `delivered` | INTEGER | `1` = delivered, `0` = pending |

**Indexes:** `idx_msg_recip (recipient_id)`, `idx_msg_sent (sent_at)`

### `heartbeat_log`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `ws_id` | TEXT | WebSocket session identifier |
| `ts` | TEXT | ISO-8601 timestamp |
| `latency_ms` | INTEGER | Round-trip latency in ms (nullable) |

---

## 8. npm / Node.js Integration

`blackroad-websocket-manager` exposes its SQLite database using a well-documented schema, making it straightforward to integrate from any language.

**Read active connections from Node.js:**

```js
const Database = require("better-sqlite3");
const path = require("path");
const os = require("os");

const dbPath = path.join(os.homedir(), ".blackroad", "websocket-manager.db");
const db = new Database(dbPath, { readonly: true });

const active = db
  .prepare("SELECT ws_id, agent, metadata FROM connections WHERE status = 'active'")
  .all();

console.log(`Active connections: ${active.length}`);
active.forEach((row) => {
  console.log(row.ws_id, row.agent, JSON.parse(row.metadata));
});
```

**Send a message from Node.js:**

```js
const { v4: uuidv4 } = require("uuid");

db.prepare(
  `INSERT INTO messages (msg_id, msg_type, sender_id, recipient_id, content, delivered)
   VALUES (?, 'data', NULL, ?, ?, 1)`
).run(uuidv4(), "ws-001", JSON.stringify({ event: "ping" }));
```

> **npm packages required:** `better-sqlite3`, `uuid`  
> Install with: `npm install better-sqlite3 uuid`

---

## 9. Stripe Webhook Integration

Use `blackroad-websocket-manager` to push real-time Stripe events to connected agents.

```python
import json
import stripe
from pathlib import Path
from src.main_module import get_db, ConnectionPool, broadcast

stripe.api_key = "sk_live_..."           # set via environment variable in production
WEBHOOK_SECRET = "whsec_..."             # set via environment variable in production

def handle_stripe_webhook(raw_body: bytes, sig_header: str) -> dict:
    """Verify a Stripe webhook and broadcast the event to all active agents."""
    event = stripe.Webhook.construct_event(raw_body, sig_header, WEBHOOK_SECRET)

    db = get_db()
    pool = ConnectionPool(db)

    payload = {
        "source": "stripe",
        "type": event["type"],          # e.g. "payment_intent.succeeded"
        "data": event["data"]["object"],
    }

    # Broadcast to all agents, or filter by agent name as needed
    delivered = broadcast(pool, db, payload, msg_type="stripe_event")
    return {"delivered_to": len(delivered), "event_type": event["type"]}
```

**Example: filter delivery to a billing agent only**

```python
delivered = broadcast(
    pool, db, payload,
    filter_fn=lambda c: c.agent == "billing",
    msg_type="stripe_event",
)
```

> **Requirements:** `pip install stripe`  
> Always load `stripe.api_key` and `WEBHOOK_SECRET` from environment variables — never hard-code credentials.

---

## 10. Production Deployment

### Running as a background service

Use your OS process manager to run the heartbeat-check sweep on a schedule:

**systemd timer (Linux):**

```ini
# /etc/systemd/system/ws-heartbeat-check.service
[Unit]
Description=BlackRoad WebSocket heartbeat sweep

[Service]
Type=oneshot
ExecStart=/usr/bin/python3 /opt/blackroad/src/main_module.py heartbeat-check --timeout 30
```

```ini
# /etc/systemd/system/ws-heartbeat-check.timer
[Unit]
Description=Run heartbeat check every 30 seconds

[Timer]
OnBootSec=30
OnUnitActiveSec=30

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now ws-heartbeat-check.timer
```

### Environment variables

| Variable | Description |
|---|---|
| `BLACKROAD_DB_PATH` | Override default SQLite path (`~/.blackroad/websocket-manager.db`) |
| `STRIPE_API_KEY` | Stripe secret key (never commit to source) |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret |

### Security checklist

- [ ] Database file permissions set to `600` (owner read/write only)
- [ ] Stripe credentials loaded from environment variables, not source code
- [ ] Webhook endpoint validates `Stripe-Signature` header before processing
- [ ] SQLite WAL mode enabled for concurrent read access: `PRAGMA journal_mode=WAL;`
- [ ] Connection metadata does not contain PII unless encrypted at rest

---

## 11. Testing

Run the full test suite:

```bash
python -m pytest tests/ -v
```

Run with linting:

```bash
pip install flake8
flake8 src/ --max-line-length=120
python -m pytest tests/ -v
```

**Test coverage includes:**

- Add / get / remove connections
- Broadcast (all connections and filtered by agent)
- Direct message delivery
- Heartbeat update and stale-connection sweep
- Message history retrieval
- Connection stats aggregation
- Message count increment

---

## 12. Contributing

This repository is proprietary software owned by BlackRoad OS, Inc. External contributions require a signed Contributor License Agreement (CLA). Please contact the maintainers before submitting pull requests.

All CI checks (lint + tests) must pass before any merge.

---

## 13. License

Copyright © 2024–2026 BlackRoad OS, Inc. All Rights Reserved.  
Founder, CEO & Sole Stockholder: Alexa Louise Amundson.

See [LICENSE](./LICENSE) for the full terms.
