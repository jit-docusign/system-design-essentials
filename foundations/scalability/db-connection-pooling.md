# Database Connection Pooling

## What it is

**Database connection pooling** is a technique for reusing a finite set of pre-established database connections rather than creating and destroying a connection for every database query. Since establishing a new database connection is expensive (network handshake, authentication, server-side memory allocation), reusing connections dramatically reduces per-query overhead and allows a much higher query rate.

Connection pooling is almost always required in production applications. Without it, concurrent requests spawn too many connections, exhausting the database's connection limit.

---

## How it works

### The Cost of a New Database Connection

Creating a new database connection is not free:
1. TCP connection (SYN/SYN-ACK/ACK handshake) — ~1ms on LAN.
2. TLS negotiation (if using SSL) — ~3–10ms.
3. Database authentication (password validation, permission lookup).
4. Server-side process/thread allocation (PostgreSQL forks a backend process per connection — expensive).
5. Connection setup memory: PostgreSQL allocates ~3–10MB per connection.

For a request that takes 5ms to execute a query, the connection setup could be 5–15ms — longer than the query itself. At high concurrency, this is untenable.

### How Pooling Works

A pool maintains N pre-established, idle connections. A query request:
1. **Borrows** a connection from the pool.
2. **Executes** the query.
3. **Returns** the connection to the pool (not closed).

```
App tier (100 concurrent requests)
      │           │           │
      ▼           ▼           ▼
   ┌────────────────────────────────┐
   │      Connection Pool           │
   │  [conn1] [conn2] [conn3] ... [connN] │
   └────────────────────────────────┘
                │
         (reused connections)
                │
              DB
           (max_connections)
```

The pool isolates the database from the application's concurrency level. 1,000 concurrent app requests can share 50 pooled connections.

### Pool Sizing

**Too few connections**: requests queue waiting for an available connection → high latency.  
**Too many connections**: DB runs out of memory and CPU for managing connections → performance degrades.

**Formula (starting point):**
```
pool_size = (core_count * 2) + effective_spindle_count

PostgreSQL recommendation: ((2 * cpu_cores) + disk_spindles)
Typical production: 20–50 connections per application server
```

*Little's Law* applied to connection pools:
```
Pool throughput = pool_size / (avg_query_time_seconds)
Pool needed     = target_QPS × avg_query_time_seconds
```

Example: 10ms avg query, target 500 QPS → pool_size = 500 × 0.010 = 5 connections theoretically. Add safety margin: 20–50 connections.

### Application-Level Pooling

Most frameworks/ORMs include built-in connection pooling:

**Python (SQLAlchemy):**
```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://user:pass@localhost/mydb",
    pool_size=20,         # base pool size
    max_overflow=10,      # extra connections beyond pool_size on spike
    pool_timeout=30,      # wait timeout before giving up
    pool_pre_ping=True,   # validate connections before use
)
```

**Node.js (pg):**
```javascript
const { Pool } = require('pg');
const pool = new Pool({
    connectionString: process.env.DATABASE_URL,
    max: 20,          // maximum connections in pool
    idleTimeoutMillis: 30000,  // close idle connections after 30s
    connectionTimeoutMillis: 2000,
});
```

**Java (HikariCP — fastest Java connection pool):**
```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost/mydb");
config.setMaximumPoolSize(30);
config.setMinimumIdle(10);
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);
config.setMaxLifetime(1800000);
HikariDataSource ds = new HikariDataSource(config);
```

### External/Proxy Pooling

When running many application servers, each with its own connection pool, the total connection count can still overwhelm the database:

```
100 app servers × 20 pool connections each = 2,000 connections to PostgreSQL
PostgreSQL max_connections = 500 → database crashes
```

**Proxy-based poolers** sit between the app tier and the database, centralizing the pool:

```
App Tier (100 servers)         Proxy Pooler          Database
  app-1 (20 conns) ──┐                           ┌── PostgreSQL
  app-2 (20 conns) ──┼──► PgBouncer (50 conns) ──┤   (50 real connections)
  ...                │                           └── handles up to 500 concurrent
  app-100(20 conns) ─┘      (2000 client conns → 50 server conns)
```

**PgBouncer (PostgreSQL)**: the standard proxy pooler for PostgreSQL. Three modes:
- **Session mode**: one server connection per client session. Simplest but least efficient pooling.
- **Transaction mode**: server connection is held only during a transaction, then returned. Most efficient — ideal for short-lived transactions.
- **Statement mode**: connection returned after each statement. Very aggressive; breaks multi-statement transactions.

**ProxySQL (MySQL)**: similar proxy for MySQL with additional query routing capabilities (read/write split, traffic mirroring).

### Connection Validation ("Ping on Borrow")

Long-idle connections may be closed by the database or a firewall without the pool knowing. When borrowed, the connection appears valid but the first query fails:

- **Validation query**: run a lightweight query (`SELECT 1`) when borrowing to verify connectivity. Adds ~0.5ms per borrow.
- **Pool pre-ping** (SQLAlchemy, HikariCP): check connection health before lending it. Reconnect if dead.
- **Maximum lifetime**: retire connections after N minutes regardless to avoid stale state.

### Connection Pool Anti-Patterns

```python
# Bad: opening a connection per request (no pooling)
@app.route('/users/<int:id>')
def get_user(id):
    conn = psycopg2.connect(DATABASE_URL)  # new connection every time!
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = ?", (id,))
    user = cursor.fetchone()
    conn.close()
    return user

# Good: use pool
@app.route('/users/<int:id>')
def get_user(id):
    with db_pool.getconn() as conn:      # borrowed from pool
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = ?", (id,))
        return cursor.fetchone()
    # connection automatically returned to pool on context exit
```

---

## Key Trade-offs

| Benefit | Trade-off |
|---|---|
| Eliminates connection setup overhead | Pool must be sized correctly (too small = queue, too large = DB pressure) |
| Caps DB connection count at pool_size | Long-running queries hold connections; slows pool availability |
| Reduces DB memory usage | Long-idle connections may become stale (need validation) |
| Absorbs burst request concurrency | Proxy poolers add another infrastructure component |

---

## When to use

Connection pooling is **always required** in production applications. The only time you can skip it:
- Single-user scripts or batch jobs that run infrequently.
- Unit test fixtures.

Even moderate web applications (100 concurrent users) require pooling. Without it, connection creation overhead and connection count limits will cause errors at scale.

---

## Common Pitfalls

- **Pool too large for the database**: sum of all application pools must not exceed `max_connections`. A 5-node app cluster × 100 pool size = 500 connections. PostgreSQL default max_connections is 100. Use PgBouncer to multiplex.
- **Not returning connections**: a query that throws an exception without returning its connection to the pool (missing try/finally or context manager) eventually exhausts the pool. All frameworks provide context managers (`with`) to handle this automatically — use them.
- **Pool exhaustion causing cascading failure**: if pool is full and timeout is long (30s), requests queue for 30 seconds before failing. Under load, the queue grows, response times spike, and the service appears down. Set shorter timeouts and alert on pool exhaustion.
- **Blocking operations inside a pooled connection**: running a 10-minute report while holding a pooled connection blocks that connection for the entire duration. Run long queries on a separate connection or use a read replica.
