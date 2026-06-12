# ACID Properties — Database Fundamentals

> Section: ACID Properties (Transactions, Atomicity, Isolation, Consistency, Durability + bonus deep dives)

---

## 1. What is a Transaction?

A **transaction** is a group of one or more database operations (queries) treated as a single unit of work. Either all of them succeed, or none do.

### Why transactions exist
SQL data is structured across multiple tables. Most real operations need more than one query to complete logically (e.g., a money transfer needs a SELECT + two UPDATEs). A transaction groups these into one indivisible operation.

### Transaction Lifecycle
```
BEGIN
  → query 1
  → query 2
  → query 3
COMMIT   (persist everything permanently)
   or
ROLLBACK (undo everything, as if it never happened)
```

- **BEGIN** — starts a new transaction
- **COMMIT** — "I'm satisfied, persist all changes to disk permanently"
- **ROLLBACK** — "Forget everything I did, undo all changes"

### Implicit vs Explicit Transactions
- **Explicit**: you write `BEGIN ... COMMIT` yourself
- **Implicit**: every single statement you run (e.g., a lone `UPDATE`) is automatically wrapped by the database in its own `BEGIN...COMMIT` (auto-commit mode)

**There is no such thing as a query outside a transaction.** You're always in one — explicit or implicit.

### Read-Only Transactions
Even SELECT-only operations benefit from transactions — they give you a **consistent snapshot** of the data at the moment the transaction started. Useful for reports/analytics where multiple queries must reflect the same point in time.

### The Engineering Trade-off (Write-as-you-go vs Write-on-commit)
| Approach | COMMIT speed | ROLLBACK speed | Crash risk |
|---|---|---|---|
| Write to disk as queries execute (Postgres) | Fast | Slower (must undo disk writes) | Smaller crash window |
| Hold in memory, flush on COMMIT | Slow (big flush) | Instant (drop memory) | Bigger crash window |

No right/wrong — pure trade-off. Postgres optimizes for fast commits.

---

## 2. Atomicity (A)

> **All queries in a transaction succeed, or none do.** The transaction is an "atom" — indivisible.

### Two failure modes
1. **Explicit query failure** — constraint violation, duplicate PK, bad SQL → entire transaction rolls back, even if 99/100 queries succeeded.
2. **Database crash mid-transaction** — no ROLLBACK was issued, the DB just died. On restart, the DB detects the incomplete transaction via its logs and **automatically rolls it back** before becoming available.

### The horror scenario (no atomicity)
```
accounts: id=1 balance=1000, id=2 balance=500
Transfer ₹100 from account 1 → account 2

UPDATE accounts SET balance=900 WHERE id=1;  ✅ done
💥 CRASH
UPDATE accounts SET balance=600 WHERE id=2;  ❌ never ran
```
Result without atomicity: account 1 = 900, account 2 = 500. ₹100 vanished — pure corruption.
Result with atomicity: DB restarts, detects uncommitted txn, rolls account 1 back to 1000.

### Real-world consequence: Long transactions are dangerous
If a DB crashes mid-transaction, on restart it **must finish rolling back before it can serve traffic**. The instructor has seen SQL Server rollbacks take **over an hour** for long transactions. CPU/memory hammered, DB effectively offline during cleanup.

**Rule: keep transactions short. Minimum queries needed. Commit fast.**

---

## 3. Isolation (I)

> Defines what an in-flight transaction can see of changes made by other concurrent transactions — committed or not.

There's no universally "correct" answer — it's a dial you tune based on your use case. Weak isolation = fast but risky. Strong isolation = safe but slow.

### The Four Read Phenomena (all undesirable)

#### 1. Dirty Read
You read data another transaction **wrote but hasn't committed**. That data might get rolled back — you acted on something that never officially existed.

```
T1: SELECT sum(qty*price) FROM sales → sees committed: 130
T2: UPDATE sales SET qty=15 (uncommitted)
T1: SELECT sum(...) → reads T2's uncommitted value → gets 155
T2: ROLLBACK  ← never happened, but T1 already used 155
```
Worst phenomenon. Virtually no database allows this by default.

#### 2. Non-Repeatable Read
You read a value. Another transaction **commits** a change to it. You read again (in the same transaction) — different result.
```
T1: reads product1=50, product2=80 (sum=130)
T2: UPDATE qty=15, COMMIT (legit commit)
T1: SELECT SUM(...) → now 155, inconsistent within T1's own transaction
```
Difference from dirty read: the data WAS legitimately committed. Problem is within-transaction inconsistency.

#### 3. Phantom Read
A range query returns new rows that **didn't exist before**, because another transaction inserted matching rows and committed.
```
T1: SELECT SUM(qty) FROM sales → 30
T2: INSERT new row matching the range, COMMIT
T1: SELECT SUM(qty) FROM sales → 40 (new row "appeared")
```
Different from non-repeatable read: you never read this row before — can't lock something that doesn't exist yet. Hence "phantom."

#### 4. Lost Update
Two transactions read the same value, both modify it, both write back — one overwrites the other.
```
Initial qty = 10
T1: reads 10, computes 10+10=20
T2: reads 10, computes 10+5=15, COMMITs → qty=15
T1: COMMITs → qty=20  ← T2's update is LOST
```
Classic race condition — silently wrong data, both transactions think they succeeded.

### Isolation Levels (the fix)

| Level | Dirty Read | Non-Repeatable | Phantom | Lost Update |
|---|---|---|---|---|
| READ UNCOMMITTED | ❌ Possible | ❌ Possible | ❌ Possible | ❌ Possible |
| READ COMMITTED (default in Postgres/Oracle/SQL Server) | ✅ Fixed | ❌ Possible | ❌ Possible | ❌ Possible* |
| REPEATABLE READ | ✅ Fixed | ✅ Fixed | ❌ Possible (MySQL/Oracle/SQL Server) / ✅ Fixed (**Postgres**) | ✅ Fixed |
| SERIALIZABLE | ✅ Fixed | ✅ Fixed | ✅ Fixed | ✅ Fixed |

*Row-level write locks prevent literal lost updates at READ COMMITTED in most DBs, but multi-step read-then-write logic can still be wrong (see below).

### How databases implement isolation

**Pessimistic (lock-based):**
- Row/page/table locks. Other transactions *wait*.
- Risk: "lock escalation" — row lock escalates to table lock, everything queues up.
- Older MySQL, SQL Server lean this way.

**Optimistic (MVCC — Multi-Version Concurrency Control):**
- No locks for reads. Each transaction gets a "snapshot" version.
- On conflict at commit time → abort with a **serialization error**, app must retry.
- Postgres: every UPDATE creates a **new row version** (old version kept until VACUUM cleans it up).
- MySQL: writes new value directly, keeps old value in a separate **undo log** — readers reconstruct old version from there (more costly for long transactions).
- NoSQL databases generally prefer optimistic — locking is expensive at scale.

### 🔑 Postgres-specific quirk (very important, common interview trap)
**In Postgres, REPEATABLE READ = snapshot isolation = also prevents phantom reads.**
In MySQL/Oracle/SQL Server, REPEATABLE READ does NOT prevent phantom reads — need SERIALIZABLE for that.

This is because Postgres's REPEATABLE READ takes a full database snapshot at `BEGIN` via MVCC — nothing committed afterward (changed rows OR new rows) is visible.

---

## 4. REPEATABLE READ vs SERIALIZABLE — Deep Dive

### Row locks ≠ full protection
Row-level locks only protect **the exact row being written, at write time**. They do NOT protect:
- Multiple reads across a transaction staying consistent
- Decisions made based on earlier reads
- Conditions/predicates (e.g., "no booking exists for room 5 at 10am") when the conflicting transactions insert *different* rows

### Why REPEATABLE READ isn't enough — Two key examples

**A) Double-booking problem (predicate conflict, no shared row)**
```
T1: SELECT COUNT(*) WHERE room=5 AND slot=10am → 0
T2: SELECT COUNT(*) WHERE room=5 AND slot=10am → 0
T1: INSERT booking (room=5, slot=10am, id=101), COMMIT ✅
T2: INSERT booking (room=5, slot=10am, id=102), COMMIT ✅ -- NO ERROR!
```
Both succeed under REPEATABLE READ — room double-booked. REPEATABLE READ only checks "did you touch a row I touched" — here, row 101 ≠ row 102, so no conflict detected. The thing that conflicts is the *condition* (predicate), not a physical row.

**B) The A/B swap problem (disjoint rows, impossible result)**
```
Table: [A, A, B, B]
T1: UPDATE SET field='B' WHERE field='A'  (touches the 2 A-rows)
T2: UPDATE SET field='A' WHERE field='B'  (touches the 2 B-rows)

Under REPEATABLE READ: both commit successfully (zero row overlap)
Result: [A, A, B, B] -- same as before, but...
```
Check what a TRUE serial execution would give:
- T1 then T2: [A,A,B,B] → all A→B → [B,B,B,B] → all B→A → [A,A,A,A]
- T2 then T1: [A,A,B,B] → all B→A → [A,A,A,A] → all A→B → [B,B,B,B]

Either serial order gives **all A's or all B's** — never a mix. But REPEATABLE READ concurrently produced `[A,A,B,B]` — a result **impossible under any serial order**. That's the bug.

### What SERIALIZABLE does differently
- **Not** "true" one-at-a-time execution (too slow) — transactions run concurrently
- Postgres uses **SSI (Serializable Snapshot Isolation)** — tracks read/write dependencies between transactions
- If it detects a cycle (the concurrent result couldn't have come from any valid serial order), it **aborts one transaction** with error code `40001` ("could not serialize access due to read/write dependencies")
- The aborted transaction must be **retried** by the application

### MVCC vs SERIALIZABLE — clarifying the relationship
- **Row versioning (MVCC)** = the mechanism that creates snapshots — used by REPEATABLE READ AND SERIALIZABLE
- **SERIALIZABLE's extra layer** = dependency/conflict detection on top of MVCC → aborts transactions when needed
- Row copies = MVCC's job. Conflict detection/abort = SERIALIZABLE's job.

### Mandatory pattern: retry logic
```javascript
async function runTransaction() {
  for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
    try {
      await db.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
      // ... queries ...
      await db.query('COMMIT');
      return;
    } catch (err) {
      if (err.code === '40001') { // serialization_failure
        await db.query('ROLLBACK');
        continue; // retry
      }
      throw err;
    }
  }
  throw new Error('Transaction failed after max retries');
}
```

### Why not just use SERIALIZABLE everywhere?
| Cost | Detail |
|---|---|
| Abort/retry rate | Scales with contention — under heavy write load, 10-30%+ aborts possible |
| Dependency tracking overhead | Extra bookkeeping/memory (SIReadLocks) even for non-conflicting transactions |
| App complexity | Retry logic everywhere — risk of bugs (infinite loops, non-idempotent side effects like double emails) |
| Unpredictable latency | p99 latency hard to reason about — txn might succeed instantly or retry 3x |

### Practical alternatives to blanket SERIALIZABLE
- **Explicit row locking**: `SELECT ... FOR UPDATE` at READ COMMITTED — surgical, predictable
- **Unique constraints**: for double-booking, `UNIQUE(room, slot)` — second INSERT just fails with constraint violation, simplest fix

### The senior-level framing
> Use the **weakest isolation level that satisfies your correctness requirements** — every level above that is wasted cost.

| Scenario | Typical choice |
|---|---|
| Simple CRUD, single-row updates | READ COMMITTED |
| Reports/analytics, multi-row consistent reads | REPEATABLE READ |
| Financial transfers, bookings, inventory with cross-row invariants | SERIALIZABLE (or explicit locking / unique constraints) |
| High-throughput, tolerant of minor inconsistency | READ COMMITTED + app-level checks |

---

## 5. Consistency (C)

> The **outcome** of Atomicity + Isolation working correctly — not a separate mechanism, but a result.

### Two distinct types — do not conflate these

#### Type 1: Consistency in Data
The data on disk obeys all defined rules: foreign keys, unique constraints, check constraints, referential integrity, application-level invariants (e.g., "likes count in pictures table = count of rows in likes table").

**Caused by:**
- Lack of Atomicity (crash mid-transaction → corrupted/orphaned data)
- Lack of Isolation (concurrent transactions producing rule-violating results)
- Missing constraints (DB doesn't enforce, app code has bugs)

**Example — Instagram model:**
```
pictures: id=1, likes=5   ← says 5
likes table: only 2 rows for picture_id=1   ← actual count is 2

likes table: Edmund liked picture_id=4
pictures table: picture 4 doesn't exist     ← orphaned record
```
Both are **data consistency violations** — the persisted data is just wrong.

#### Type 2: Consistency in Reads
After a transaction commits a value, does a subsequent read immediately see it? This is a **distributed systems** problem — only emerges with replicas/multiple instances.

```
t=0: Write X to Primary, COMMIT ✅
t=1: Read from Replica 1 → still shows old value Z ❌
```

### Synchronous vs Asynchronous Replication
| | Sync | Async |
|---|---|---|
| Speed | Slower (waits for replica ack) | Faster (ack immediately) |
| Consistency | Strong — replica always current | Eventual — replica catches up later |
| Use case | Banking, anything that can't tolerate staleness | Social feeds, most general apps |

---

## 6. Eventual Consistency — The Full Picture

### "Data in two places = inconsistent (eventually consistent)"
The moment data exists in more than one place — replicas, caches (Redis/Memcached), search indexes — there's a window of staleness. **This applies to BOTH relational and NoSQL databases.** It's not a NoSQL-specific concept; it's a consequence of horizontal scaling/caching, period.

```
Single DB instance → consistent trivially
Add a replica      → write to leader, replica briefly stale → eventually consistent
Add a cache        → write to DB, cache briefly stale → eventually consistent
```

### Does staleness matter? (Engineering judgment call)

| Low cost of staleness | High cost of staleness |
|---|---|
| Like/view counts | Account balances |
| Follower counts | Inventory counts (overselling risk) |
| Recommendation feeds | Auth/permission checks (security hole if stale) |
| "Last seen" timestamps | Idempotency keys (duplicate charges) |

Real risk example: double-spend — if a balance check reads a stale (higher) value across replicas, a user could withdraw the same money twice before replicas sync.

### 🔑 THE MOST IMPORTANT DISTINCTION: Eventual Consistency ≠ Fix for Corruption

> **Eventual consistency ONLY applies to read-consistency (replication lag).**
> **It does NOT apply to data-consistency (corruption from broken atomicity).**

```
Read consistency issue:
  Leader=X, Replica=Z (stale) → wait → Replica=X → SELF-RESOLVES
  "Eventual consistency" = valid description ✅

Data consistency issue (corruption):
  Transaction should update 7 tables, crashed after 3
  4 tables permanently out of sync
  → waiting does NOTHING — this is corruption, not staleness
  "Eventual consistency" ≠ valid description ❌
```

If atomicity/isolation/durability are broken, **no amount of "eventual consistency" will fix it**. That's a bug requiring repair, not a designed trade-off resolving itself.

---

## 7. Durability (D)

> **Committed = permanent.** A crash, power loss, or reboot after COMMIT must not lose that data.

### The core trade-off
Disk writes are slow. Making every write durable (synced to disk) slows everything down. Databases play games with this:
- Write to memory first, flush to disk occasionally → faster writes, risk of loss on crash
- Strong relational DBs (Postgres, etc.) prioritize durability for committed transactions

### Durability Technique 1: Write-Ahead Log (WAL)
Instead of updating full data structures (tables, B-trees, indexes) on every commit — record only the **delta/change** in a compact, sequential, append-only log.

```
Commit → write WAL entry (sequential, fast) → ack "committed"
       → actual table/index updates can happen lazily afterward

Crash recovery → replay WAL → reconstruct consistent state
```
WAL is the source of truth; actual table files are a materialized view of it. Postgres calls it WAL; MySQL calls it the redo log.

### Durability Technique 2: Async Snapshots (Redis-style)
All writes → RAM (fast) → background process periodically snapshots to disk.
- Fast writes, but data since last snapshot is lost on crash.
- Redis lets you configure snapshot frequency — explicit speed-vs-durability dial.
- AOF (Append-Only File) in Redis ≈ same concept as WAL.

### The OS Cache Problem (subtle but critical)
```
DB: "OS, write this WAL entry to disk"
OS: writes to RAM cache instead, says "done!" (lying — batches for performance)
DB: "Great, committed."

💥 Power cut → RAM cache lost → "committed" data never reached disk → DATA LOST
```

### The Fix: fsync
A system call that bypasses OS cache and forces a real physical write to disk.
- Guarantees durability
- **Slow** — this is why commits have a measurable performance cost
- Redis offers configurable durability (e.g., "lose up to 3 seconds of writes for faster throughput") — almost no relational DB gives you this dial; relational DBs prioritize durability by default.

### When is sacrificing durability acceptable?
Caching layers, approximate analytics, IoT telemetry, session stores — places where losing a few seconds of data is tolerable and write speed matters more. NOT acceptable: financial records, medical data, anything where every write matters.

---

## 8. ACID — How It All Connects

```
ATOMICITY    → "All or nothing" — partial transactions never persist
     ↓
ISOLATION    → "Transactions don't bleed into each other" — concurrency safety
     ↓
CONSISTENCY  → "Rules always hold" — the OUTCOME of A + I working correctly
     ↓
DURABILITY   → "Committed = permanent" — survives crashes
```

They're not independent — A and I protect C, and D ensures the consistent state survives beyond the transaction. A database missing any one of the four is not truly ACID compliant.

---

---

# ⚡ QUICK REVISION — Fast Interview Prep

## One-Page Cheat Sheet

| Property | Guarantee | Breaks when | Key mechanism |
|---|---|---|---|
| **Atomicity** | All queries succeed or none do | Crash mid-transaction, query failure | Rollback logs, crash recovery on restart |
| **Isolation** | Concurrent transactions don't interfere | Weak isolation level + concurrent access | Locks (pessimistic) or MVCC (optimistic) |
| **Consistency** | Data follows defined rules; reads reflect commits | A or I breaks (data); replication lag (reads) | Constraints + FK + (A+I) for data; sync/async replication for reads |
| **Durability** | Committed data survives crashes | OS cache lies, async-only writes | WAL + fsync |

## Read Phenomena — Quick Match

| Phenomenon | One-liner |
|---|---|
| **Dirty Read** | Reading another txn's **uncommitted** write |
| **Non-Repeatable Read** | Same row, read twice in one txn, value changed (because other txn **committed**) |
| **Phantom Read** | Range query returns a **new row** that didn't exist on first read |
| **Lost Update** | Two txns read-modify-write same value; one overwrites the other silently |

## Isolation Level Quick Table

| Level | Fixes |
|---|---|
| READ UNCOMMITTED | Nothing |
| READ COMMITTED (default) | Dirty reads |
| REPEATABLE READ | + Non-repeatable reads, lost updates. Phantoms: fixed in **Postgres only** (MVCC snapshot) |
| SERIALIZABLE | Everything, including predicate/dependency conflicts. Costs: aborts/retries (`40001`) |

## Postgres-Specific Facts (high-value interview signal)
- REPEATABLE READ in Postgres = snapshot isolation = ALSO fixes phantom reads (unique to Postgres)
- SERIALIZABLE = SSI (Serializable Snapshot Isolation) — dependency cycle detection, not literal serial execution
- MVCC = row versioning mechanism (used by both REPEATABLE READ & SERIALIZABLE)
- SERIALIZABLE's extra layer = conflict detection + abort, ON TOP OF MVCC

## The "Trap" Questions
1. **"Does REPEATABLE READ prevent phantom reads?"** → Depends on DB. Postgres: yes. MySQL/Oracle/SQL Server: no.
2. **"Is eventual consistency a NoSQL-only thing?"** → No — applies anytime data exists in 2+ places (replicas, caches), regardless of DB type.
3. **"Will eventual consistency fix a corrupted multi-table write?"** → No. Eventual consistency = read-lag (self-resolving). Corruption from broken atomicity = permanent until manually repaired.
4. **"Why not use SERIALIZABLE everywhere if it's the safest?"** → Abort/retry overhead scales with contention; use the weakest level that satisfies correctness needs.
5. **"Row locks prevent all concurrency issues?"** → No — they only protect the exact row at write time. Don't protect multi-row predicates (double-booking) or multi-step read-then-decide logic.

---

## Interview Pointers — How to Answer

**"Explain ACID"**
Walk through in order A → I → C → D (not alphabetical) — explain that C is the *outcome* of A+I, not a separate mechanism. Shows deeper understanding than a rote definition.

**"What is a transaction?"**
Unit of work, BEGIN/COMMIT/ROLLBACK, mention implicit auto-commit transactions — most candidates forget this part.

**"Explain isolation levels"**
Don't just list them — explain the *read phenomenon* each level fixes, and call out the Postgres REPEATABLE READ exception. This signals real depth.

**"Design a booking system — how do you prevent double-booking?"**
Mention: (1) why REPEATABLE READ fails (predicate conflict, no shared row), (2) SERIALIZABLE + retry as one option, (3) UNIQUE constraint as the simplest practical fix, (4) SELECT FOR UPDATE as a locking alternative. Shows you can weigh trade-offs, not just recite theory.

**"What is eventual consistency? When is it acceptable?"**
Define it correctly (read-lag across replicas/caches, applies to SQL & NoSQL), then immediately pivot to the cost-of-staleness framework — likes vs balances. Then mention the corruption distinction unprompted — this is the differentiator most candidates miss.

**"What is WAL and why does it matter?"**
Append-only log of deltas written before main data structures update. Enables fast commits (sequential writes) + crash recovery (replay log). Mention fsync and the OS cache problem if you want to go deeper.

**General tip:** When asked "what database would you use for X," always frame your answer as a **trade-off**, not a recommendation. "It depends on whether X needs strong consistency or can tolerate eventual consistency, and whether writes or reads dominate" — this framing alone signals seniority.

---

*End of ACID Properties section. Next section: [to be added]*
