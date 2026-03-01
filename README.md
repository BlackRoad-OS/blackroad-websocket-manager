# blackroad-websocket-manager

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/downloads/)
[![Tests](https://img.shields.io/badge/tests-passing-brightgreen.svg)](#testing)
[![npm](https://img.shields.io/badge/npm-compatible-red.svg)](#npm-cli-wrapper)

Production-grade WebSocket connection tracker for **BlackRoad OS** agents.  
Manages connections, broadcasts messages, tracks heartbeats, and integrates with
Stripe webhook events — all backed by a local SQLite store with automatic indexing.

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Installation](#installation)
   - [Python (pip)](#python-pip)
   - [npm CLI Wrapper](#npm-cli-wrapper)
4. [Configuration](#configuration)
5. [CLI Reference](#cli-reference)
   - [connect](#connect)
   - [disconnect](#disconnect)
   - [list](#list)
   - [broadcast](#broadcast)
   - [send](#send)
   - [heartbeat](#heartbeat)
   - [heartbeat-check](#heartbeat-check)
   - [history](#history)
   - [stats](#stats)
6. [Python API Reference](#python-api-reference)
   - [get_db](#get_db)
   - [ConnectionPool](#connectionpool)
   - [add_connection](#add_connection)
   - [remove_connection](#remove_connection)
   - [broadcast](#broadcast-1)
   - [send_message](#send_message)
   - [heartbeat_check](#heartbeat_check)
   - [get_message_history](#get_message_history)
   - [connection_stats](#connection_stats)
7. [Stripe Webhook Integration](#stripe-webhook-integration)
8. [Database Schema](#database-schema)
9. [Testing](#testing)
10. [Contributing](#contributing)
11. [License](#license)

---

## Overview

`blackroad-websocket-manager` is the authoritative connection registry for
BlackRoad OS agents.  Every agent opens a WebSocket, registers itself via
`connect`, and from that moment on can receive targeted or broadcast messages
from any other service in the platform — including real-time Stripe payment
events.

The library is intentionally dependency-light: the only runtime requirement is
the Python standard library.  Persistence is handled by SQLite with pre-built
indexes for high-throughput reads.

---

## Features

| Category | Detail |
|---|---|
| **Connection lifecycle** | Register, retrieve, and gracefully disconnect WebSocket sessions |
| **In-memory pool** | O(1) lookups backed by a persistent SQLite store |
| **Broadcast** | Fan-out to all connections or to a filtered subset by agent name or custom predicate |
| **Direct messaging** | Point-to-point delivery with delivery tracking |
| **Heartbeat monitoring** | Automatic removal of stale connections after a configurable timeout |
| **Message history** | Full audit trail in the `messages` table with delivery flags |
| **Stripe events** | Drop-in handler to forward `payment_intent.*` and other Stripe webhook events to connected agents |
| **Indexed SQLite** | `connections`, `messages`, and `heartbeat_log` tables — each with purpose-built indexes |

---

## Installation

### Python (pip)

```bash
pip install blackroad-websocket-manager
```

Requirements: **Python 3.9+**.  No third-party runtime dependencies.

For Stripe webhook integration, install the optional extra:

```bash
pip install "blackroad-websocket-manager[stripe]"
# installs stripe>=7.0
```

### npm CLI Wrapper

A thin npm wrapper exposes the CLI to Node.js / JavaScript projects:

```bash
npm install --save-dev blackroad-websocket-manager
# or globally
npm install -g blackroad-websocket-manager
```

After installation the `ws-manager` binary is available on your `PATH`:

```bash
ws-manager connect octavia --ws-id ws-001
```

The npm package delegates every command to the bundled Python runtime, so
Python 3.9+ must be present on the host.

---

## Configuration

| Environment Variable | Default | Description |
|---|---|---|
| `BLACKROAD_WS_DB` | `~/.blackroad/websocket-manager.db` | Absolute path to the SQLite database file |
| `BLACKROAD_HB_TIMEOUT` | `30` | Default heartbeat timeout in seconds |
| `STRIPE_WEBHOOK_SECRET` | *(required for Stripe)* | Stripe webhook signing secret (`whsec_…`) |

The `--db` CLI flag overrides `BLACKROAD_WS_DB` for a single invocation.

---

## CLI Reference

All commands share the global `--db <path>` flag to point at a custom database.

### connect

Register a new WebSocket connection for an agent.

```bash
ws-manager connect <agent> [--ws-id <id>] [--metadata '<json>']
```

| Argument | Required | Description |
|---|---|---|
| `agent` | Yes | Logical agent name (e.g. `octavia`, `cipher`) |
| `--ws-id` | No | Explicit WebSocket ID; auto-generated (UUID v4) if omitted |
| `--metadata` | No | JSON string of arbitrary key/value pairs |

```bash
ws-manager connect octavia --ws-id ws-001 --metadata '{"ip":"10.0.0.1","region":"us-west"}'
# Connected: ws-001 (agent=octavia)
```

### disconnect

Mark a connection as disconnected.

```bash
ws-manager disconnect <ws_id>
```

### list

Print all currently active connections.

```bash
ws-manager list
```

### broadcast

Send a message to all active connections, optionally filtered by agent name.

```bash
ws-manager broadcast '<message>' [--agent <name>] [--type <msg_type>]
```

```bash
# Broadcast a deploy event to every agent
ws-manager broadcast '{"event":"deploy","version":"2.1.0"}'

# Broadcast only to octavia instances
ws-manager broadcast '{"event":"alert"}' --agent octavia
```

### send

Send a direct message to a single connection by WebSocket ID.

```bash
ws-manager send <ws_id> '<message>'
```

```bash
ws-manager send ws-001 '{"ping":true}'
# Sent message <uuid> to ws-001
```

### heartbeat

Record a heartbeat (keepalive) for a connection.

```bash
ws-manager heartbeat <ws_id> [--latency <ms>]
```

### heartbeat-check

Scan all connections and remove any whose heartbeat is older than `--timeout`
seconds (default: 30).

```bash
ws-manager heartbeat-check [--timeout <seconds>]
```

### history

Display the message history for a connection or for the entire pool.

```bash
ws-manager history [--ws-id <id>] [--limit <n>]
```

### stats

Print a JSON summary of pool and message statistics.

```bash
ws-manager stats
# {
#   "active_connections": 3,
#   "total_ever_connected": 10,
#   "total_messages": 47,
#   "agents": {"octavia": 2, "cipher": 1}
# }
```

---

## Python API Reference

**Installed via pip** (`pip install blackroad-websocket-manager`):

```python
# The pip package name uses hyphens; the Python import name uses underscores.
from blackroad_websocket_manager import (
    get_db, ConnectionPool,
    add_connection, remove_connection,
    broadcast, send_message,
    heartbeat_check, get_message_history,
    connection_stats,
)
```

**Using the source tree directly** (development / no pip install):

```python
import sys, pathlib
sys.path.insert(0, str(pathlib.Path("src")))
from main_module import (
    get_db, ConnectionPool,
    add_connection, remove_connection,
    broadcast, send_message,
    heartbeat_check, get_message_history,
    connection_stats,
)
```

### get_db

```python
get_db(db_path: Path = DB_PATH) -> sqlite3.Connection
```

Open (or create) the SQLite database, run schema migrations, and return the
connection.  Safe to call multiple times — `CREATE TABLE IF NOT EXISTS` guards
are in place for all tables.

### ConnectionPool

```python
pool = ConnectionPool(db_conn)
```

In-memory registry of active connections, hydrated from SQLite on startup.

| Method | Signature | Description |
|---|---|---|
| `add` | `(connection: Connection) -> Connection` | Persist and register a connection |
| `remove` | `(ws_id: str) -> bool` | Mark disconnected and evict from pool |
| `get` | `(ws_id: str) -> Optional[Connection]` | O(1) lookup by WebSocket ID |
| `get_all` | `() -> List[Connection]` | All active connections |
| `count` | `() -> int` | Number of active connections |
| `update_heartbeat` | `(ws_id, latency_ms=None) -> bool` | Record keepalive and log to `heartbeat_log` |
| `increment_message_count` | `(ws_id: str) -> None` | Atomically increment the per-connection message counter |

### add_connection

```python
add_connection(pool, ws_id, agent, metadata=None) -> Connection
```

### remove_connection

```python
remove_connection(pool, ws_id) -> bool
```

### broadcast

```python
broadcast(pool, db_conn, message, filter_fn=None, msg_type="broadcast", sender_id=None) -> List[str]
```

Fan-out `message` to every connection that satisfies `filter_fn` (all
connections when `None`).  Each delivery is recorded in the `messages` table.
Returns the list of `ws_id`s that received the message.

```python
db = get_db()
pool = ConnectionPool(db)

add_connection(pool, "ws-001", "octavia", {"region": "us-west"})
add_connection(pool, "ws-002", "cipher",  {"region": "eu-central"})

# Broadcast to everyone
delivered = broadcast(pool, db, {"event": "ping"})
print(f"Delivered to {len(delivered)} connections")

# Broadcast only to agents in us-west
delivered = broadcast(
    pool, db,
    {"event": "maintenance"},
    filter_fn=lambda c: c.metadata.get("region") == "us-west",
)
```

### send_message

```python
send_message(pool, db_conn, ws_id, message, msg_type="data", sender_id=None) -> Optional[Message]
```

Deliver a message to a single connection.  Returns `None` if `ws_id` is not
found in the active pool.

### heartbeat_check

```python
heartbeat_check(pool, db_conn, timeout=30) -> dict
# Returns {"active": [...], "timed_out": [...]}
```

### get_message_history

```python
get_message_history(db_conn, ws_id=None, limit=50) -> List[dict]
```

### connection_stats

```python
connection_stats(pool, db_conn) -> dict
```

---

## Stripe Webhook Integration

`blackroad-websocket-manager` can relay Stripe webhook events to any connected
agent in real time.  A typical integration looks like this:

### 1. Install the Stripe extra

```bash
pip install "blackroad-websocket-manager[stripe]"
```

### 2. Configure environment variables

```bash
export STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxx
export BLACKROAD_WS_DB=/var/blackroad/ws.db
```

### 3. Forward Stripe events to connected agents

```python
import stripe
import os
from blackroad_websocket_manager import get_db, ConnectionPool, broadcast

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]
WEBHOOK_SECRET = os.environ["STRIPE_WEBHOOK_SECRET"]

def handle_stripe_webhook(request_body: bytes, stripe_signature: str) -> None:
    """Call this from your web framework's webhook endpoint."""
    event = stripe.Webhook.construct_event(
        request_body, stripe_signature, WEBHOOK_SECRET
    )

    db = get_db()
    pool = ConnectionPool(db)

    # Relay the raw Stripe event to every active agent
    broadcast(
        pool, db,
        {"source": "stripe", "type": event["type"], "data": event["data"]},
        msg_type="stripe_event",
    )
```

### 4. Targeting specific agents

```python
# Only notify billing agents about payment events
if event["type"].startswith("payment_intent."):
    broadcast(
        pool, db,
        {"source": "stripe", "type": event["type"], "data": event["data"]},
        filter_fn=lambda c: c.agent == "billing",
        msg_type="stripe_payment",
    )
```

### Supported Stripe event types

| Event | Recommended `msg_type` |
|---|---|
| `payment_intent.succeeded` | `stripe_payment` |
| `payment_intent.payment_failed` | `stripe_payment` |
| `customer.subscription.created` | `stripe_subscription` |
| `customer.subscription.deleted` | `stripe_subscription` |
| `invoice.paid` | `stripe_invoice` |
| `invoice.payment_failed` | `stripe_invoice` |

Any Stripe event type can be forwarded; the table above lists the most common
ones used in production BlackRoad OS deployments.

---

## Database Schema

The SQLite database is stored at `~/.blackroad/websocket-manager.db` by default.

### `connections`

| Column | Type | Description |
|---|---|---|
| `id` | `INTEGER PK` | Auto-increment primary key |
| `ws_id` | `TEXT UNIQUE` | WebSocket identifier |
| `agent` | `TEXT` | Agent name |
| `metadata` | `TEXT` | JSON blob of arbitrary key/value pairs |
| `connected_at` | `TEXT` | ISO-8601 UTC timestamp |
| `last_heartbeat` | `TEXT` | ISO-8601 UTC timestamp of most recent keepalive |
| `status` | `TEXT` | `active` \| `disconnected` |
| `message_count` | `INTEGER` | Cumulative messages delivered to this connection |
| `disconnected_at` | `TEXT` | ISO-8601 UTC timestamp; `NULL` while active |

**Indexes:** `idx_conn_agent (agent)`, `idx_conn_status (status)`

### `messages`

| Column | Type | Description |
|---|---|---|
| `id` | `INTEGER PK` | Auto-increment primary key |
| `msg_id` | `TEXT UNIQUE` | UUID v4 message identifier |
| `msg_type` | `TEXT` | Message type label (e.g. `broadcast`, `stripe_payment`) |
| `sender_id` | `TEXT` | Originating WebSocket ID or `NULL` for system messages |
| `recipient_id` | `TEXT` | Destination WebSocket ID |
| `content` | `TEXT` | JSON-serialised message payload |
| `sent_at` | `TEXT` | ISO-8601 UTC timestamp |
| `delivered` | `INTEGER` | `1` = delivered, `0` = pending |

**Indexes:** `idx_msg_recip (recipient_id)`, `idx_msg_sent (sent_at)`

### `heartbeat_log`

| Column | Type | Description |
|---|---|---|
| `id` | `INTEGER PK` | Auto-increment primary key |
| `ws_id` | `TEXT` | WebSocket identifier |
| `ts` | `TEXT` | ISO-8601 UTC timestamp of the keepalive |
| `latency_ms` | `INTEGER` | Round-trip latency in milliseconds; `NULL` if not measured |

---

## Testing

```bash
# Install test dependencies
pip install pytest

# Run the full test suite
python -m pytest tests/ -v

# Run with coverage
pip install pytest-cov
python -m pytest tests/ -v --cov=src --cov-report=term-missing
```

All tests use an in-memory (temp-path) SQLite database and are fully isolated —
no external services or network access required.

---

## Contributing

1. Fork the repository and create a feature branch.
2. Make your changes with tests for any new behaviour.
3. Run `python -m pytest tests/ -v` and confirm all tests pass.
4. Open a pull request against `main`.

Please follow existing code style: standard library only, type annotations on
all public functions, and docstrings for every public class and function.

---

## License

[MIT](LICENSE) © BlackRoad OS
