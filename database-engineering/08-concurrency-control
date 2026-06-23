# Section 7: Concurrency Control

> Source: Hussein Nasser — "Fundamentals of Database Engineering"
> Priority: 🔴 CORE | 📺 Must Watch (double booking demo + connection pooling benchmark)

---

## 1. Exclusive Lock vs. Shared Lock

### The two lock types

**Exclusive lock (write lock)**
Acquired when you want to **write/update** a value. You are the only connection allowed to touch that value — nobody can read or write it while you hold the lock.
- Acquired by: `UPDATE`, `INSERT`, `DELETE`, or explicitly via `SELECT FOR UPDATE`
- Rule: can only be acquired if **no shared locks AND no other exclusive lock** exist on that value

**Shared lock (read lock)**
Acquired when you want to **read** a value and need it to stay unchanged for the duration of your transaction. Many connections can hold shared locks on the same value simultaneously.
- Acquired by: long-running reads, reports, aggregations
- Rule: can only be acquired if **no exclusive lock** exists on that value

### Mutual exclusion rules

```
Multiple shared locks:     ✅ allowed simultaneously (many readers, no writer)
Shared + exclusive:        ❌ never (reader blocks writer OR writer blocks reader)
Two exclusive locks:       ❌ never (only one writer at a time)
```

**Mental model: readers block writers, writers block everyone.**

### Worked example: Alice, Bob, Charlie (banking)

```
Alice:   UPDATE balance (+$200) → exclusive lock → succeeds (no other locks) → commits → lock released
Alice:   Long reporting job → shared lock on her balance → nobody can write her balance now
Bob:     Long reporting job → shared lock on his balance → succeeds (shared locks coexist)
Charlie: Transfer $300 to Bob → needs exclusive lock on Bob's balance → FAILS (Bob holds shared lock)
Bob:     Finishes report → releases shared lock
Charlie: Retries exclusive lock → succeeds → transfer completes ✅
```

### Real-world implications

**Banking:** never want balance changing mid-read during a transaction. Shared locks enforce stability.

**Configuration systems:** acquiring exclusive lock during config update blocks all readers until write commits → every reader always gets latest config, never stale.

**"Bank of America midnight" phenomenon:** large reporting/reconciliation jobs acquire shared locks at scale, blocking all write transactions. Bank deliberately chose consistency over availability during that window.

### Advantages and disadvantages

| | Detail |
|---|---|
| ✅ Consistency | Reads see stable values; writes don't race with concurrent reads |
| ✅ Isolation | Mechanical implementation of the I in ACID (Section 1) |
| ❌ Concurrency suffers | Locks serialize access — transactions queue or fail under contention |
| ❌ Long transactions costly | A long-held shared lock blocks all writers until released |

---

## 2. Two-Phase Locking (2PL)

### The core idea

2PL governs **when** locks are acquired and released — in exactly two phases:

**Phase 1 — Growing phase (acquire only)**
Transaction acquires all locks it needs. Can acquire new locks, cannot release any.

**Phase 2 — Shrinking phase (release only)**
Transaction begins releasing locks (on commit or rollback). Cannot acquire any new locks after this phase starts.

```
Phase 1 (Growing):   ACQUIRE → ACQUIRE → ACQUIRE
                                                ↓
Phase 2 (Shrinking):                     RELEASE → RELEASE → RELEASE
```

**The critical rule:** once you release even one lock, you can never acquire another. This one-way gate prevents a class of anomalies caused by interleaved lock acquisition/release.

In Postgres: **commit or rollback triggers Phase 2** — all locks released simultaneously. You can't release individual locks mid-transaction under standard 2PL.

---

## 3. The Double Booking Problem

### Why it happens — the check-then-act race condition

The naive booking implementation:

```javascript
// BROKEN — race condition possible
const client = await pool.connect();
await client.query('BEGIN');

// Step 1: check availability
const result = await client.query(
    'SELECT * FROM seats WHERE id = $1 AND is_booked = 0',
    [seatId]
);

if (result.rowCount === 0) {
    await client.query('ROLLBACK');
    return res.status(400).send('Seat already booked');
}

// Step 2: book it
await client.query(
    'UPDATE seats SET is_booked = 1, name = $1 WHERE id = $2',
    [name, seatId]
);

await client.query('COMMIT');
res.send('Booked successfully');
```

**The race condition:**
```
T1 (Rick):    SELECT → seat available ✅ → [paused before UPDATE]
T2 (Edmund):  SELECT → seat available ✅ → UPDATE → COMMIT ✅
T1 (Rick):    [resumes] → UPDATE → COMMIT ✅
```

Both transactions read "available" before either writes. Both pass the check. Both execute the UPDATE. Both get "Booked successfully." **One seat, two confirmed bookings.** Last write wins — but both users get a success confirmation. This is the classic **check-then-act race condition**.

Any gap between "check if available" and "mark as taken" is a window for a race condition.

---

## 4. The Fix — `SELECT FOR UPDATE` (Row-Level Exclusive Lock)

```javascript
// CORRECT — race-safe
const client = await pool.connect();
try {
    await client.query('BEGIN');

    // Check AND lock in one atomic operation
    const result = await client.query(
        'SELECT * FROM seats WHERE id = $1 AND is_booked = 0 FOR UPDATE',
        [seatId]
    );

    if (result.rowCount === 0) {
        await client.query('ROLLBACK');
        return res.status(400).json({ error: 'Seat already booked' });
    }

    await client.query(
        'UPDATE seats SET is_booked = 1, name = $1 WHERE id = $2',
        [name, seatId]
    );

    await client.query('COMMIT');
    res.json({ success: true });
} catch (e) {
    await client.query('ROLLBACK');
    throw e;
} finally {
    client.release();
}
```

**What `FOR UPDATE` does:** acquires an exclusive lock on every row returned by the SELECT. Lock held until COMMIT or ROLLBACK.

**Fixed flow:**
```
T1 (Rick):    SELECT FOR UPDATE → acquires exclusive lock on row ✅
T2 (Melissa): SELECT FOR UPDATE → 🔴 BLOCKS — waits for T1's lock

T1 (Rick):    UPDATE → COMMIT → lock released
T2 (Melissa): unblocked → SELECT result: is_booked = 1 → rowCount = 0
              → "Seat already booked" ✅
```

T2 is forced to wait until T1 fully commits. By the time T2 reads the row, it sees T1's committed state. **Double booking is structurally impossible.**

### Row-level vs table-level lock

`FOR UPDATE` locks at the **row level**, not the table level:
- Two users booking **different seats** → no blocking, full concurrency ✅
- Two users booking the **same seat** → one blocks until other commits ✅

### `FOR UPDATE` variants

```sql
-- Error immediately if lock can't be acquired (fail fast)
SELECT * FROM seats WHERE id = $1 FOR UPDATE NOWAIT;

-- Skip locked rows entirely (good for job queues)
SELECT * FROM seats WHERE id = $1 FOR UPDATE SKIP LOCKED;
```

`NOWAIT` is generally better for booking UX — immediately tell the user "seat is being booked, try another" rather than making them wait indefinitely.

### Database support

| Database | `FOR UPDATE` | Timeout option |
|---|---|---|
| Postgres | ✅ | `NOWAIT` / `SKIP LOCKED` — no configurable wait time natively |
| MySQL/InnoDB | ✅ | `WAIT n` / `NOWAIT` |
| Oracle | ✅ | `WAIT n` timeout supported natively |

---

## 5. Alternative: Update-With-Condition (and Why to Be Careful)

Some engineers skip the SELECT entirely and bake the condition into the UPDATE:

```sql
-- T1
UPDATE seats SET is_booked = 1, name = 'Hussein'
WHERE id = 1 AND is_booked = 0;
-- check rowCount: 1 = success, 0 = already booked

-- T2 (concurrent)
UPDATE seats SET is_booked = 1, name = 'Rick'
WHERE id = 1 AND is_booked = 0;
-- blocks... then gets rowCount = 0 ✅
```

**Does it work? Yes in Postgres — but you need to understand why, because it's not guaranteed everywhere.**

### What happens under the hood (Postgres-specific)

1. T1 executes UPDATE → B+ tree traversal on `id` index → finds tuple → goes to heap
2. T1 acquires **implicit exclusive lock** on the row (UPDATE always locks implicitly)
3. T2 executes UPDATE → finds same tuple → tries to acquire exclusive lock → **blocked**
4. T1 commits → lock released, `is_booked = 1` committed
5. T2 unblocks → **Postgres re-evaluates the WHERE clause against committed values**
6. New evaluation: `is_booked = 0` is now FALSE → UPDATE affects 0 rows → rowCount = 0 ✅

**The re-evaluation on unblock is the key Postgres behavior** — and it's not universal.

### Why this is database-dependent

| Database | Behavior on lock release |
|---|---|
| **Postgres** | Re-evaluates WHERE against committed values (READ COMMITTED) → rowCount = 0 ✅ |
| **MySQL/InnoDB** | Different internal implementation — not guaranteed same result |
| **SQL Server** | Checks in memory — may not re-evaluate the same way |

**Dangerous edge case even in Postgres:** if you have a composite index on `(id, is_booked)`, Postgres might find matching values in the index without going to the heap — and the index might still show the old `is_booked = 0` value. In that scenario, Postgres might incorrectly proceed without the heap re-check.

### `SELECT FOR UPDATE` vs. Update-with-condition

| | `SELECT FOR UPDATE` | Update-with-condition |
|---|---|---|
| **Lock type** | Explicit user lock | Implicit DB-internal lock |
| **Control** | Full — you control what happens between lock and commit | Limited — at mercy of DB re-evaluation semantics |
| **Multi-step transactions** | ✅ Easy — lock held across all steps | ❌ Hard — only covers the single UPDATE |
| **Portability** | ✅ Consistent across DBs | ⚠️ Behavior varies by DB and index configuration |
| **Isolation level** | Works at REPEATABLE READ and above | Requires READ COMMITTED for re-evaluation |

**Recommendation:** prefer `SELECT FOR UPDATE`. More explicit, more portable, more control, easier to reason about.

---

## 6. Pessimistic vs. Optimistic Concurrency Control

Two broader philosophies for handling concurrent access:

**Pessimistic concurrency control** (Postgres, Oracle default)
- Assume conflicts will happen → take a lock upfront → protect the resource
- "Always take the umbrella even if it might not rain"
- Pros: no retry logic needed, failures happen for the right reasons, predictable
- Cons: lock overhead, potential blocking under contention

**Optimistic concurrency control** (common in NoSQL, distributed systems)
- Assume conflicts are rare → don't lock upfront → at commit time, detect if anything changed, fail if so → application retries
- Pros: no locking overhead during transaction, better for low-contention scenarios
- Cons: transaction can fail at commit time → retry logic required in app; under high contention, retry storms occur

**Nasser's preference:** pessimistic — "if my transaction fails, I want it to fail for the right reasons, not because of a concurrency problem I have to handle in application code."

---

## 7. SQL OFFSET — Why It's Slow and What to Use Instead

### What OFFSET actually does

Most developers think `OFFSET 100 LIMIT 10` means "skip to row 100, fetch 10." What actually happens:

**Fetch 110 rows, discard the first 100, return 10.**

The database has no way to "jump" to row 100 — it must read and process every row up to that point. The discarded rows are full reads — index traversal, heap access, all wasted.

**Cost scales linearly with offset:**

| Query | Rows DB processes | Rows returned | Time (cached) |
|---|---|---|---|
| `OFFSET 0 LIMIT 10` | 10 | 10 | 0.2ms |
| `OFFSET 1,000 LIMIT 10` | 1,010 | 10 | 1ms |
| `OFFSET 100,000 LIMIT 10` | 100,010 | 10 | 79ms |
| `OFFSET 1,000,000 LIMIT 10` | 1,000,010 | 10 | 620ms+ |

**Lock escalation note:** SQL Server escalates to table-level locks when touching this many rows — effectively locking the table for everyone. Postgres doesn't do lock escalation, but the IO/CPU waste is still severe.

### The second problem: duplicate rows on insertion

Offset-based pagination has a correctness bug, not just a performance bug:

```
User reads page 10 (OFFSET 100): sees rows 101–110
Someone inserts a new record near row 101
User reads page 11 (OFFSET 110): row 110 (previously seen) shifts into the result set again
→ duplicate row the user already saw
```

**Real-world consequence:** infinite-scroll feeds showing the same article twice; paginated tables with missing or repeated records when data changes between page loads.

### The fix: Keyset Pagination (Cursor-Based Pagination)

Instead of "give me rows at position N," use "give me rows after ID X":

```sql
-- Page 1: no cursor
SELECT id, title
FROM news
ORDER BY id DESC
LIMIT 10;
-- Returns IDs: 1000, 999, 998... 991
-- Client saves last seen ID: 991

-- Page 2: use last seen ID as cursor
SELECT id, title
FROM news
WHERE id < 991
ORDER BY id DESC
LIMIT 10;
-- Returns IDs: 990, 989... 981
-- Client saves new cursor: 981

-- Page 1000 (deep): same cost as page 1
SELECT id, title
FROM news
WHERE id < [cursor]
ORDER BY id DESC
LIMIT 10;
```

**What the database does:**
1. B+ tree traversal on `id` index → finds cursor position in O(log n) — Section 4
2. Walks leaf nodes forward (leaf linking — Section 4) to collect exactly 10 rows
3. **Total rows processed: always 10, regardless of how deep the page is**

```
EXPLAIN ANALYZE:
OFFSET approach (page 100,000): rows processed = 1,000,010
Keyset approach (page 100,000): rows processed = 10
```

### Keyset vs. OFFSET tradeoffs

| | OFFSET | Keyset |
|---|---|---|
| **Performance** | Degrades linearly with depth | Constant — O(log n) always |
| **Correctness** | Duplicates/gaps on insert/delete | Stable — cursor anchors to specific row |
| **Jump to arbitrary page** | ✅ Easy | ❌ Must walk pages sequentially |
| **Implementation** | Simple | Client must track cursor |
| **Insert-safe** | ❌ Breaks pagination order | ✅ Unaffected |
| **UI pattern** | Numbered pages ("Page 5 of 47") | Infinite scroll / next-previous |

**When to still use OFFSET:** small tables (<10K rows), admin interfaces where users jump to arbitrary page numbers and data rarely changes. Don't over-engineer for 500 rows.

**When keyset is essential:** high-traffic APIs, millions of rows, infinite-scroll feeds, any deep pagination pattern.

### `FETCH FIRST n ROWS ONLY` — SQL standard alternative to LIMIT

```sql
SELECT title FROM news
ORDER BY id DESC
FETCH FIRST 10 ROWS ONLY;
```

Functionally identical to `LIMIT 10` — just the SQL standard syntax (Postgres, Oracle, SQL Server). Same performance.

---

## 8. Connection Pooling

### The problem: connection-per-request

The naive approach — connect on every request, disconnect when done:

```javascript
app.get('/users', async (req, res) => {
    const client = new Client({ host, port, user, password });
    await client.connect();      // TCP 3-way handshake + auth — expensive
    const result = await client.query('SELECT * FROM users');
    await client.end();          // teardown — expensive
    res.json(result.rows);
});
```

Every request pays: TCP handshake + Postgres auth handshake + query + teardown. On local DB: ~40ms overhead. On remote cloud DB: significantly worse (network round-trips for each handshake).

### Connection pooling — the fix

Create a **pool** of pre-established connections on server startup. Requests borrow a connection, use it, return it — no handshake overhead per request.

**Benchmark (1000 requests):**
- Connection per request: ~40ms average
- Pool: ~19ms average — ~50% faster locally. Even larger improvement on remote databases.

### Setup — one pool for the entire app

```javascript
// db.js — created ONCE, exported, imported wherever needed
import { Pool } from 'pg';

const pool = new Pool({
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    max: 10,                        // max simultaneous connections
    connectionTimeoutMillis: 5000,  // wait max 5s if all connections busy (0 = forever — dangerous)
    idleTimeoutMillis: 10000,       // destroy idle connections after 10s (0 = never)
});

export default pool;
```

Node.js module system caches imports — `db.js` runs exactly once no matter how many files import it. Every route that imports `pool` gets the same object in memory.

```
src/
├── db.js              ← pool created here, once
├── app.js             ← server startup
└── routes/
    ├── users.js       ← imports pool from db.js
    └── orders.js      ← imports pool from db.js
```

### When to use what

#### `pool.query()` — 90% of cases (simple CRUD)

Any single statement. Connection borrowed and returned automatically. No `client.release()` needed.

```javascript
// Simple read
const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [id]
);

// Simple write
await pool.query(
    'INSERT INTO orders (user_id, total) VALUES ($1, $2)',
    [userId, total]
);
```

#### `pool.connect()` — transactions (multiple statements, atomic)

Any time multiple queries must succeed or fail together. A transaction's `BEGIN` → queries → `COMMIT` must all run on the **same connection**. `pool.query()` might route each statement to a different connection — the transaction would be lost.

```javascript
const client = await pool.connect();
try {
    await client.query('BEGIN');
    await client.query(
        'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
        [amount, fromId]
    );
    await client.query(
        'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
        [amount, toId]
    );
    await client.query('COMMIT');
} catch (e) {
    await client.query('ROLLBACK');
    throw e;
} finally {
    client.release(); // MANDATORY — always in finally
}
```

#### `pool.connect()` + `FOR UPDATE` — contested resources

```javascript
const client = await pool.connect();
try {
    await client.query('BEGIN');
    const result = await client.query(
        'SELECT * FROM seats WHERE id = $1 AND is_booked = 0 FOR UPDATE',
        [seatId]
    );
    if (result.rowCount === 0) {
        await client.query('ROLLBACK');
        return res.status(400).json({ error: 'Seat already booked' });
    }
    await client.query(
        'UPDATE seats SET is_booked = 1, name = $1 WHERE id = $2',
        [name, seatId]
    );
    await client.query('COMMIT');
} catch (e) {
    await client.query('ROLLBACK');
    throw e;
} finally {
    client.release(); // MANDATORY
}
```

### Why `client.release()` is mandatory for `pool.connect()`

Forgetting `client.release()` leaks the connection permanently out of the pool:

```
Request 1: pool.connect() → connection A leaked (no release)
Request 2: pool.connect() → connection B leaked
...
Request 10: pool.connect() → all 10 connections leaked
Request 11, 12, 13... → hang forever waiting for a connection that never returns
```

No crash, no error — just silent hanging requests and climbing response times. `finally` guarantees release even if an error is thrown mid-transaction.

### Decision tree

```
One statement?
└── pool.query()  ← no release needed

Multiple statements that must be atomic?
└── pool.connect() + BEGIN/COMMIT/ROLLBACK + client.release() in finally
    │
    ├── Race condition risk on a specific row?
    │   └── Add SELECT FOR UPDATE before your UPDATE
    │
    └── No race condition?
        └── Just BEGIN/queries/COMMIT
```

### Summary table

| Scenario | Pattern | `client.release()` needed? | Use for |
|---|---|---|---|
| Per-request connect | `new Client()` per req | `client.end()` | ❌ Never |
| Single client startup | `new Client()` on start | No | Scripts/tooling only |
| Simple CRUD | `pool.query()` | No — automatic | ✅ Default for all reads/writes |
| Transaction | `pool.connect()` | **YES — in finally** | Multi-statement atomic ops |
| Lock + transaction | `pool.connect()` + `FOR UPDATE` | **YES — in finally** | Contested resources |

### Production considerations

**Pool size is not "bigger = better":** each connection consumes ~5–10MB RAM on Postgres server + a file descriptor + a background worker. Postgres default max connections = 100. A pool of 100 on a small instance overwhelms it. Common rule: `pool size = DB CPU cores × 2 + small buffer`. **PgBouncer** is a dedicated connection pooler proxy used in production to manage this at scale.

**`connectionTimeoutMillis: 0` (wait forever) is dangerous:** under extreme load, requests pile up waiting for a connection, memory grows unbounded, server crashes. Set a finite timeout (e.g., 5000ms) and return a 503 when pool is exhausted rather than queuing indefinitely.

---

## Glossary

| Term | Definition |
|---|---|
| **Exclusive lock (write lock)** | Lock that gives one transaction sole access to a row — no other reads or writes allowed while held. |
| **Shared lock (read lock)** | Lock that allows many readers simultaneously but blocks any writer. |
| **Two-phase locking (2PL)** | Protocol where locks are only acquired in Phase 1 (growing) and only released in Phase 2 (shrinking). Once a lock is released, no new locks can be acquired. |
| **`SELECT FOR UPDATE`** | SQL syntax that acquires an exclusive row-level lock on selected rows, held until COMMIT or ROLLBACK. |
| **`FOR UPDATE NOWAIT`** | Variant that errors immediately if the lock can't be acquired, instead of waiting. |
| **`FOR UPDATE SKIP LOCKED`** | Variant that skips already-locked rows instead of waiting — useful for job queues. |
| **Check-then-act race condition** | Bug where two transactions both check a condition (seat available), both pass, both act — leading to double booking or similar corruption. |
| **Row-level lock** | Lock applied to a specific row only, not the whole table — allows concurrent access to other rows. |
| **Lock escalation** | When a DB upgrades many row-level locks to a table-level lock (SQL Server does this; Postgres does not). |
| **Pessimistic concurrency** | Assume conflicts will happen — lock upfront before acting. |
| **Optimistic concurrency** | Assume conflicts are rare — act without locking, detect conflict at commit time, retry if needed. |
| **OFFSET pagination** | Fetch and discard the first N rows to reach a page. Cost scales linearly with page depth. |
| **Keyset pagination** | Use the last seen row's ID as a cursor — `WHERE id < cursor LIMIT n`. Constant cost regardless of depth. |
| **Cursor** | The last seen ID/value passed back to the client for keyset pagination — anchors the next page to a specific row. |
| **Connection pool** | A pre-established set of database connections shared across all requests — eliminates per-request handshake overhead. |
| **`pool.query()`** | Executes a single statement using any available pool connection — auto-releases. |
| **`pool.connect()`** | Borrows a dedicated connection from the pool — must be manually released via `client.release()`. |
| **`client.release()`** | Returns a borrowed connection back to the pool. Forgetting this leaks connections permanently. |
| **PgBouncer** | Dedicated Postgres connection pooler proxy — sits between app and DB, manages connections at scale. |
| **`connectionTimeoutMillis`** | How long to wait if all pool connections are busy before erroring. |
| **`idleTimeoutMillis`** | How long an unused connection sits in the pool before being destroyed. |

---

## Quick Revision (skim before an interview)

1. Exclusive lock = one writer, no readers/writers allowed simultaneously. Shared lock = many readers allowed, no writers.
2. Shared locks block exclusive locks. Exclusive locks block everyone.
3. 2PL: Phase 1 = acquire only. Phase 2 = release only. Once you release, you can never acquire again. Commit/rollback triggers Phase 2.
4. `SELECT FOR UPDATE` acquires an exclusive row-level lock at read time — closes the window between "check" and "act." This is how double booking is prevented.
5. `FOR UPDATE NOWAIT` errors immediately if lock unavailable. `SKIP LOCKED` skips locked rows — useful for job queues.
6. Update-with-condition (`UPDATE ... WHERE is_booked = 0`) works in Postgres due to WHERE re-evaluation on unblock — but is database-implementation-dependent and less controllable than `FOR UPDATE`.
7. Pessimistic = lock upfront (Postgres default). Optimistic = no lock, detect conflict at commit, retry (NoSQL common).
8. `OFFSET n LIMIT 10` makes the DB read and discard n rows every time. Cost is O(n). Also causes duplicate rows when data changes between pages.
9. Keyset pagination (`WHERE id < cursor LIMIT 10`) uses the B+ tree to jump directly to the cursor position — constant cost O(log n) regardless of page depth.
10. One pool per app, not per route or per request. `pool.query()` for single statements (no release needed). `pool.connect()` for transactions (always `client.release()` in `finally`).

---

## Interview Questions & Answers

**Q1: What's the difference between an exclusive lock and a shared lock?**
A: An exclusive lock (write lock) gives one transaction sole ownership of a row — no other transaction can read or write it until the lock is released. A shared lock (read lock) allows many transactions to read the same value simultaneously but blocks any writer from acquiring an exclusive lock. Multiple shared locks can coexist; a shared lock and an exclusive lock cannot — readers block writers, and writers block everyone.

**Q2: What is two-phase locking and why does it exist?**
A: 2PL is a concurrency control protocol that divides a transaction's lock lifecycle into two phases: a growing phase where only lock acquisition is allowed, and a shrinking phase (triggered by commit or rollback) where only lock release happens. Once a lock is released, no new locks can be acquired. This protocol prevents a class of anomalies caused by interleaved lock acquisition and release across concurrent transactions, ensuring serializability.

**Q3: What is the double booking problem and how do you fix it?**
A: The double booking problem occurs when two concurrent transactions both check a resource (e.g., a seat) as available, both pass the check, and both write a booking — leading to two confirmed bookings for the same seat. The fix is `SELECT FOR UPDATE`, which acquires an exclusive row-level lock at read time. The second transaction blocks on the lock until the first commits, then re-reads the now-committed state (seat taken) and correctly rejects the booking. The key insight: the check and the lock must be one atomic operation — any gap between "check" and "act" is a race condition window.

**Q4: What does `SELECT FOR UPDATE` actually do at the database level?**
A: It performs a normal SELECT but additionally acquires an exclusive row-level lock on every returned row. The lock is held until the transaction commits or rolls back (Phase 2 of 2PL). Any other transaction attempting to acquire an exclusive or shared lock on those rows will block until the lock is released. It only locks the specific rows returned — other rows in the same table remain fully accessible, preserving concurrency for non-contested resources.

**Q5: Why is `OFFSET` pagination slow, and what's the alternative?**
A: `OFFSET n LIMIT 10` forces the database to fetch and physically discard the first n rows before returning your 10 — cost is O(n), scaling linearly with page depth. At offset 1,000,000 the DB processes 1,000,010 rows to return 10. OFFSET also causes duplicate rows when records are inserted between page loads. The alternative is keyset (cursor-based) pagination: `WHERE id < [last_seen_id] LIMIT 10`. This uses a B+ tree index to jump directly to the cursor position in O(log n), then reads exactly 10 rows via leaf linking — constant cost regardless of how deep the page is.

**Q6: What is connection pooling and why does it matter?**
A: Connection pooling maintains a pre-established set of database connections (a pool) that are reused across requests. Without pooling, each request establishes a new TCP connection (3-way handshake + Postgres auth handshake) and tears it down after — expensive overhead even before the query runs. A pool eliminates this per-request overhead: connections are already open and authenticated, requests borrow one, use it, and return it. Benchmark result: ~50% faster on local DB, even more on remote. Also prevents connection exhaustion — the pool queues requests rather than opening unlimited connections.

**Q7: When do you use `pool.query()` vs `pool.connect()` in Node.js?**
A: `pool.query()` for any single statement — it automatically borrows a connection, executes, and returns it. No `client.release()` needed. `pool.connect()` when you need a transaction (multiple statements that must be atomic) — because a transaction's BEGIN/queries/COMMIT must all run on the same connection, and `pool.query()` might route each statement to a different connection. After `pool.connect()`, `client.release()` in a `finally` block is mandatory — forgetting it permanently leaks connections out of the pool, eventually hanging all subsequent requests.

**Q8: What happens if you forget `client.release()` in a connection pool?**
A: The connection is permanently leaked out of the pool. Each forgotten `release()` removes one connection from availability. Once all connections are leaked, every subsequent `pool.connect()` call hangs indefinitely waiting for a connection that never returns. No error is thrown, no crash occurs — just silently climbing response times and eventually timeouts everywhere. This is why `client.release()` must always be in a `finally` block, which executes whether the transaction succeeds, fails, or throws an unexpected error.

**Q9: What is the difference between pessimistic and optimistic concurrency control?**
A: Pessimistic concurrency control assumes conflicts will happen and acquires locks upfront to prevent them — the default approach in Postgres and Oracle. No retry logic needed; transactions fail for legitimate reasons (resource actually taken). Optimistic concurrency control assumes conflicts are rare — transactions proceed without locking, but at commit time the DB detects if any read data was modified by another transaction and fails the transaction if so, requiring the application to retry. Optimistic is better for low-contention scenarios; pessimistic is more predictable under high contention. NoSQL databases commonly use optimistic approaches; traditional relational DBs default to pessimistic.

**Q10: Why does the update-with-condition approach work in Postgres but may not work in other databases?**
A: When `UPDATE ... WHERE id = 1 AND is_booked = 0` is blocked by another transaction's lock and then unblocks after that transaction commits, Postgres re-evaluates the WHERE clause against the newly committed row values (READ COMMITTED isolation). If `is_booked` is now 1, the condition fails and 0 rows are updated — correctly preventing double booking. MySQL and SQL Server handle lock release differently and may not perform this re-evaluation. Additionally, even in Postgres, if a composite index covers both columns, Postgres might evaluate the condition from the index (which may still show the old value) without going to the heap for re-evaluation — making the behavior unreliable. `SELECT FOR UPDATE` is preferred because it's explicit, portable, and gives full control over the transaction flow.

**Q11: What are `FOR UPDATE NOWAIT` and `FOR UPDATE SKIP LOCKED` used for?**
A: `FOR UPDATE NOWAIT` errors immediately if the lock can't be acquired instead of waiting — useful for booking systems where you want to immediately tell the user "someone else is booking this seat, try another" rather than queueing them indefinitely. `FOR UPDATE SKIP LOCKED` skips rows that are already locked and returns the next available unlocked rows — particularly useful for job queues where multiple workers compete to claim tasks: each worker grabs unlocked work items without blocking on items already claimed by other workers.

**Q12: Why should you never use `SELECT *` in production queries?**
A: `SELECT *` fetches all columns including wide ones (BLOBs, large TEXT fields) that you likely don't need — wasting IO, network bandwidth, and memory. It also breaks when table schema changes (new columns automatically appear in results, potentially breaking application logic). Always specify exactly the columns you need. This is why keyset pagination examples use `SELECT id, title` rather than `SELECT *`.
