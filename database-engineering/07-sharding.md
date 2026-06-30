# Section 6: Database Sharding

> Source: Hussein Nasser — "Fundamentals of Database Engineering"
> Priority: 🔴 CORE | 📺 Must Watch (consistent hashing explanation + "when to shard" segment especially)

---

## 1. What is Sharding?

**Sharding** = splitting a large table into partitions (shards) distributed across **multiple, independent database servers** — done to scale the system horizontally beyond what a single server can handle.

Like horizontal partitioning, sharding splits data by rows using a **shard key** (also called **partition key**) — a column whose value determines which server a given row lives on. Unlike partitioning, the routing logic lives in the **application layer or a proxy**, not inside the database engine.

### The motivating problem

A single database server hits a hard ceiling:
- Table grows → index grows deeper (B+ tree levels increase — Section 4) → queries slow down even with indexes
- More concurrent connections → server RAM, CPU, file descriptors overwhelmed
- No amount of vertical scaling (bigger machine) solves this indefinitely

Sharding breaks the ceiling by spreading data and load across N independent servers.

---

## 2. Partitioning vs. Sharding — The Precise Distinction

| | Horizontal Partitioning | Sharding |
|---|---|---|
| **Data location** | Same database server | Different database servers entirely |
| **What changes** | Table name / schema | The server/instance you connect to |
| **Table name** | Changes — e.g. `customers_west`, `customers_east` | Stays the same — `customers` on every shard |
| **Client awareness** | Zero (Postgres native) or knows table names (manual) | Must know which server to connect to |
| **Managed by** | DB engine (transparent) or client | Always client or proxy layer |
| **Transactions** | Fully supported (same DB) | Cross-shard transactions nearly impossible |
| **Joins** | Normal SQL joins work | Cross-shard joins essentially impossible |

**One-line interview answer:** "Partitioning splits a table on the **same server** — routing is DB-managed, client is agnostic. Sharding splits across **different servers** — the client or proxy must handle routing, and ACID guarantees break across shards."

---

## 3. Consistent Hashing

The core algorithm that makes sharding work for arbitrary key types (strings, UUIDs — not just numeric ranges).

### The core guarantee

Given the same input, a consistent hashing function **always returns the same server**. Deterministic, every time.

```
hash("input1") → Server 5432  (always)
hash("input2") → Server 5433  (always)
hash("input3") → Server 5434  (always)
hash("input2") → Server 5433  (always — same input, same output)
```

### One concrete implementation (naive hash % N)

```
1. Take the input string (e.g. "5ftJ2")
2. Convert characters → ASCII codes → binary → decimal number
3. decimal % N  (N = number of shards) → remainder 0 to N-1
4. Map remainder to a specific server
```

**Example (3 shards):**
```
hash("5ftJ2") → number 10
10 % 3 = 1
1 → Server 5433 (base 5432 + 1)
```

### The hash ring

Consistent hashing is also called a **hash ring** — inputs and servers are mapped onto a circular ring of hash values. To find which server owns an input: hash the input, find where it lands on the ring, walk clockwise to the nearest server node.

**Why a ring instead of just hash % N:**

Naive `hash % N` breaks catastrophically when N changes:
- 3 servers: `hash("5ftJ2") % 3 = 1` → Server 5433
- Add a 4th server: `hash("5ftJ2") % 4 = 2` → now Server 5434
- **Almost every key remaps to a different server** → massive data migration required

A proper hash ring: when you add or remove a server, **only the keys previously assigned to that server need remapping** — everything else stays put. This makes adding/removing shards operationally safe.

**Who uses this:** Cassandra uses a consistent hash ring as its core data distribution mechanism. Every node owns a range of the ring; data is routed deterministically by partition key hash.

---

## 4. Sharding Demo — URL Shortener (Node.js + 3 Postgres Shards)

### Schema (identical on every shard)

```sql
-- init.sql (auto-executed on container startup via /docker-entrypoint-initdb.d/)
CREATE TABLE url_table (
    id      SERIAL NOT NULL PRIMARY KEY,
    url     TEXT,
    url_id  CHAR(5)
);
```

Same table name, same schema on all three shards — this is the defining sharding characteristic. The server changes, the schema doesn't.

### Infrastructure setup

```dockerfile
# Dockerfile — custom Postgres image with schema pre-loaded
FROM postgres
COPY init.sql /docker-entrypoint-initdb.d/
```

```bash
docker build -t pg_shard .

docker run --name pg_shard_1 -p 5432:5432 -d pg_shard
docker run --name pg_shard_2 -p 5433:5432 -d pg_shard
docker run --name pg_shard_3 -p 5434:5432 -d pg_shard
```

Three independent Postgres instances, same schema, different ports (simulating different servers).

### Application code (cleaned up)

```javascript
const { Client } = require('pg');
const HashRing = require('hashring');       // use hashring, not consistent-hash (buggy)
const crypto = require('crypto');
const express = require('express');
const app = express();

// Three shard connections — keyed by port
const clients = {
    '5432': new Client({ host: 'localhost', port: 5432, user: 'postgres', password: 'postgres', database: 'postgres' }),
    '5433': new Client({ host: 'localhost', port: 5433, user: 'postgres', password: 'postgres', database: 'postgres' }),
    '5434': new Client({ host: 'localhost', port: 5434, user: 'postgres', password: 'postgres', database: 'postgres' }),
};

// Hash ring with 3 nodes
const hr = new HashRing(['5432', '5433', '5434']);

async function connect() {
    await clients['5432'].connect();
    await clients['5433'].connect();
    await clients['5434'].connect();
}
connect();

// POST — create a shortened URL
app.post('/', async (req, res) => {
    const url = req.query.url;

    // 1. Hash the URL to generate a short code
    const hash = crypto.createHash('sha256').update(url).digest('base64');
    const url_id = hash.substring(0, 5);   // first 5 chars

    // 2. Consistent hash → which shard?
    const server = hr.get(url_id);         // e.g. '5433'

    // 3. Insert into the correct shard only
    await clients[server].query(
        'INSERT INTO url_table (url, url_id) VALUES ($1, $2)',
        [url, url_id]
    );

    res.json({ url_id, server });
});

// GET — resolve a short code to a full URL
app.get('/:url_id', async (req, res) => {
    const url_id = req.params.url_id;

    // Same hash ring → same shard (no re-hashing the original URL needed)
    const server = hr.get(url_id);
    const result = await clients[server].query(
        'SELECT url FROM url_table WHERE url_id = $1',
        [url_id]
    );

    if (result.rows.length > 0) {
        res.json({ url: result.rows[0].url, server });
    } else {
        res.sendStatus(404);
    }
});

app.listen(8081);
```

### Key concepts the demo demonstrates

**1. Client owns the routing logic**
`hr.get(url_id)` is the routing decision — it lives in the application, not the database. This is the fundamental difference from Postgres native partitioning.

**2. Determinism is everything**
Same `url_id` → same `hr.get()` output → same server, on every write and every read. No need to know "where did I store this?" — just re-hash and land on the same shard automatically.

**3. Shards are completely independent**
Each Postgres instance has zero knowledge of the others. No coordination, no cross-shard communication. The application is the only thing that knows all three exist.

**4. Library bug — and what it reveals**
The `consistent-hash` npm library was returning different servers for the same input on different calls — breaking the core determinism guarantee. Switched to `hashring`. This exposes a critical failure mode: if routing isn't truly deterministic, writes go to one shard and reads query a different shard → "not found" errors even though the data exists. Silent, hard-to-debug data routing failures.

**5. The resharding problem**
The demo uses a fixed 3-shard setup. Adding a 4th shard remaps nearly all keys — existing data stays on old shards, new routing sends reads to different shards → data appears "lost." Resharding requires coordinated data migration + code changes simultaneously.

---

## 5. When to Shard — The Optimization Ladder

> "Sharding is the last thing you want to do. There are so many other things you can optimize first."

Most engineers shard too early. The correct approach is to exhaust this ladder in order:

### Step 1: Indexes
Add indexes on queried columns. Zero architectural change. Limitation: as the table grows, the index grows — eventually approaching a full index scan even with indexes.

### Step 2: Horizontal Partitioning
Split the table into partitions on the same server. DB handles routing transparently. Transactions still work. Joins still work. Smaller indexes per partition (Section 4 fan-out math). Client doesn't change. **Fixes slow queries on large tables with near-zero complexity.**

### Step 3: Read Replicas (Replication)
If the problem is too many concurrent read connections overwhelming the server (not slow queries), add read replicas. One master handles writes; replicas handle reads. A proxy (Nginx, HAProxy, PgBouncer) routes reads to replicas. No application-level sharding complexity. **Fixes read scalability without touching ACID guarantees.**

Most applications are read-heavy — this alone solves the majority of scale problems.

### Step 4: Caching (Redis)
Add a cache layer for hot/repeated reads. Dramatically reduces database load. Tradeoff: cache invalidation complexity, potential read inconsistency.

### Step 5: Multi-master Replication by Region
If writes are genuinely high and geographically partitioned (US-East vs US-West customers), run two masters in separate regions. Each handles writes for its region. Conflicts are rare since regions rarely overlap. Far simpler than full sharding.

### Step 6 (Last Resort): Sharding
Only after all the above are exhausted. Even YouTube ran a single MySQL server for years, progressively adding replicas and optimizations, before being forced into sharding — and then built Vitess specifically to hide the complexity.

**Ask yourself honestly:** Is the bottleneck reads? → Replicas. Slow queries? → Indexes + partitioning. Truly saturated write server after all else is tried? → Consider sharding.

---

## 6. Pros and Cons

### Pros

**1. Horizontal scalability**
Break the single-server ceiling — data, RAM, CPU, disk IO, connection limits all spread across N machines. Instead of one extremely expensive server, N cheaper servers handle the load.

**2. Security / access control**
Shard-level isolation: VIP customer data on a dedicated shard with stricter access rules; regional data sovereignty (EU customer data physically on EU servers — GDPR compliance); different services granted access to only specific shards.

**3. Smaller, faster indexes**
Each shard holds 1/N of total rows → each shard's B+ tree is shallower → fewer IOs per lookup → index fits more easily in shard's buffer cache (RAM). Same fan-out math from Section 4: 1B rows / 10 shards = 100M rows per shard — potentially one full tree level shallower.

### Cons

**1. Client complexity / coupling**
Application must own routing logic. Every service, every engineer, every new query pattern must understand the sharding topology. Coupling the application to database topology is exactly what good software design tries to avoid. Resharding forces coordinated changes across all clients simultaneously.

**2. Cross-shard transactions are extremely hard**
Transactions within one shard = full ACID. Cross-shard = distributed transaction problem:
- Insert on shard 1 + insert on shard 2 atomically? Requires two-phase commit — complex and slow.
- Rollback on shard 1 after shard 2 already committed? Manual compensating transactions needed.
- **In practice: ACID guarantees are gone for cross-shard operations.**

**3. Schema changes must be applied to every shard**
Add a column, create an index, change a type — run the migration on every shard individually. Miss one → schema inconsistency → app breaks for data on that shard. Rolling migrations create windows where shards have different schemas — app code must handle both simultaneously.

**4. Cross-shard joins are effectively impossible**
No native SQL JOIN across database instances. Fetching from multiple shards and merging in application code is the only option — complex and slow.
*This is a genuine advantage of partitioning over sharding:* all partitions live in the same DB, so SQL joins work normally.

**5. Non-shard-key queries hit all shards (scatter-gather)**
Queries filtering on a column that isn't the shard key have no routing information — the engine must query all shards and merge results.
*Example:* customer table sharded by `customer_id`, query `WHERE name = 'Alice'` → hits all N shards → N round-trips. Potentially worse than an unsharded table.
This is why shard key selection is the most critical sharding design decision — it must match your most frequent, performance-critical query patterns.

**6. Resharding is a production nightmare**
Change shard ranges → almost every key remaps to a different server → existing data on old shards, new routing sends reads elsewhere → data appears "lost." Requires coordinated data migration + routing code changes simultaneously. In practice, teams almost never reshard after initial deployment — shard sizing must be right upfront.

---

## 7. Vitess — Middleware Sharding

**Vitess** = open-source middleware layer (originally built by YouTube) that sits in front of MySQL, parses SQL queries, and handles shard routing transparently.

- Application sends normal SQL queries — no sharding logic in app code
- Vitess parses the query, determines which shard(s) to hit, routes transparently
- Application becomes shard-agnostic — same benefit as Postgres native partitioning, but for a multi-server sharded setup
- Kubernetes-native, used in production at YouTube/Google scale

**Cost:** Vitess is itself a complex system to operate and maintain. Worth it only if you've genuinely exhausted the optimization ladder and truly need sharding's scale.

**Why YouTube built it:** they had application-level sharding for years (every service knew about shards). Changing shard ranges meant updating every service. Vitess decoupled the application from shard topology, restoring agility.

---

## Glossary

| Term | Definition |
|---|---|
| **Sharding** | Splitting a table across multiple independent database servers, with routing managed by the client/proxy. |
| **Shard** | One physical slice of a sharded dataset — a complete, independent database instance. |
| **Shard key / Partition key** | The column used to determine which shard a row belongs to. |
| **Consistent hashing** | A deterministic function that maps any input to the same server every time; minimizes key remapping when shards are added/removed. |
| **Hash ring** | The circular ring structure used in consistent hashing; inputs and servers both mapped to ring positions. |
| **Scatter-gather** | Querying all shards and merging results — required when the query doesn't filter on the shard key. |
| **Resharding** | Changing shard boundaries — requires data migration + client routing code changes simultaneously. Extremely disruptive. |
| **Read replica** | A copy of the master database that serves read-only queries — scales read throughput without sharding complexity. |
| **Vitess** | Open-source MySQL sharding middleware (built by YouTube) that handles routing transparently, decoupling apps from shard topology. |
| **Two-phase commit** | A distributed transaction protocol to achieve atomicity across multiple independent databases. Complex and slow. |
| **Vertical scaling** | Making one server bigger (more RAM, CPU, disk). Has a hard ceiling. |
| **Horizontal scaling** | Adding more servers. Sharding enables this for databases. |

---

## Quick Revision (skim before an interview)

1. Sharding = horizontal partitioning across **different servers**. Partitioning = same server. Table name stays same in sharding; server changes.
2. Client/proxy owns routing in sharding. DB engine owns routing in native partitioning.
3. Consistent hashing: same input → always same server. Determinism is the whole point.
4. Naive `hash % N` breaks when N changes — nearly all keys remap. Hash ring fixes this: only keys on the affected server need remapping.
5. ACID is effectively gone for cross-shard operations. This is the biggest practical cost of sharding.
6. Cross-shard joins are essentially impossible in native SQL.
7. Non-shard-key queries = scatter-gather = hit all shards = potentially worse than unsharded.
8. Resharding is a production nightmare — get shard sizing right upfront.
9. The optimization ladder before sharding: indexes → partitioning → read replicas → caching → multi-master by region → sharding (last resort).
10. Vitess = middleware that makes sharding transparent to the application (MySQL). Removes coupling but adds its own operational complexity.

---

## Interview Questions & Answers

**Q1: What is database sharding and why would you use it?**
A: Sharding splits a large table across multiple independent database servers, using a shard key to determine which server holds each row. It's used when a single server can no longer handle the data volume, query load, or write throughput — even after exhausting partitioning, read replicas, and caching. The main benefit is horizontal scalability: instead of one large server, you distribute data and load across N servers.

**Q2: What's the difference between sharding and horizontal partitioning?**
A: Partitioning splits a table into multiple physical tables on the **same server** — the database engine handles routing transparently, the client is fully agnostic, and ACID transactions still work. Sharding also splits the table but across **different servers** — the client or proxy must handle routing, and ACID guarantees break for cross-shard operations. Table name stays the same across shards; the server you connect to changes.

**Q3: What is consistent hashing and why is it needed for sharding?**
A: Consistent hashing is a function that deterministically maps any input (string, UUID, integer) to the same server every time. It's needed because naive hashing (`hash % N`) breaks when you add or remove a server — almost every key remaps to a different server, requiring a massive data migration. A consistent hash ring minimizes remapping: when a server is added or removed, only the keys previously assigned to that server need to move; everything else stays put.

**Q4: Why are cross-shard transactions so hard?**
A: A database transaction on one shard uses the standard ACID guarantee — if anything fails, the whole transaction rolls back atomically. A transaction spanning two shards involves two completely independent database instances. Achieving atomicity across both requires a distributed protocol (two-phase commit) — complex, slow, and a common source of subtle bugs. In practice, cross-shard transactions are considered effectively impossible in most sharding setups, which means ACID guarantees are lost for any operation that touches multiple shards.

**Q5: What is a scatter-gather query and when does it occur?**
A: A scatter-gather query occurs when the query doesn't filter on the shard key — the routing layer has no information about which shard holds the relevant data, so it must query all N shards simultaneously and merge results. Example: table sharded by `customer_id`, query `WHERE name = 'Alice'` → hits all shards. This is N database round-trips instead of 1, potentially slower than an unsharded table. It's why shard key selection must match your most frequent query patterns.

**Q6: When should you actually shard your database?**
A: Sharding should be the last resort, after exhausting: (1) indexes, (2) horizontal partitioning on the same server, (3) read replicas for read-heavy workloads, (4) caching (Redis), (5) multi-master replication for geographically partitioned writes. Each step before sharding is significantly less complex and preserves ACID. Shard only when you've genuinely saturated all these options. Most applications never reach the point where sharding is truly necessary.

**Q7: Why is resharding so painful?**
A: Resharding means changing the shard boundaries — e.g., going from 3 shards to 4, or changing range sizes. With naive `hash % N`, adding a shard remaps almost every key to a different server. Existing data stays on old shards but the new routing logic sends reads to different shards — data appears "lost" from the application's perspective. Fixing it requires coordinated data migration across all shards AND updating every service's routing logic simultaneously. In practice, most teams design their initial shard count conservatively and never reshard.

**Q8: What is Vitess and what problem does it solve?**
A: Vitess is open-source middleware (originally built by YouTube) that sits in front of MySQL and handles shard routing transparently. The application sends normal SQL queries; Vitess parses them, determines which shard(s) to hit, and routes accordingly. This decouples the application from shard topology — the app becomes shard-agnostic, just like with Postgres native partitioning. The cost is that Vitess itself is a complex system to operate. Worth it only at genuine YouTube/Google scale.

**Q9: What is the impact of shard key selection on performance?**
A: The shard key determines both data distribution and query routing efficiency. A good shard key: (1) distributes rows evenly across shards to avoid hotspots, (2) matches the column your most frequent queries filter on (avoiding scatter-gather), and (3) is immutable (changing a row's shard key value means moving it between servers). A poor shard key — one that doesn't match query patterns or distributes unevenly — can make sharding actively worse than a single unsharded server for common queries.

**Q10: What advantages does partitioning have over sharding?**
A: Several significant ones: (1) Full ACID transactions across all partitions — same database, same transaction scope. (2) SQL joins work normally across partitions. (3) Schema changes apply to one database in one migration. (4) DB engine handles routing transparently — no client-side routing logic. (5) No resharding problem — partition boundaries can be changed with a migration. The cost is that partitioning can't scale beyond one server's capacity — but that ceiling is much higher than most applications need, especially combined with read replicas.

**Q11: How does sharding improve index performance?**
A: Each shard holds 1/N of total rows, so each shard's B+ tree index is shallower than an index spanning the full dataset (directly from Section 4's fan-out math). A shallower tree means fewer IOs per lookup and a smaller index that fits more easily in the shard's buffer cache (RAM). Example: 1 billion rows across 10 shards = 100M rows per shard — potentially one full tree level shallower, meaning one fewer IO on every single index lookup.

**Q12: What failure mode occurs if your consistent hashing function isn't truly deterministic?**
A: If the routing function returns different servers for the same input on different calls, writes go to one shard and reads query a different shard — resulting in "not found" errors even though the data exists. This is silent data routing failure: no exception, no crash, just phantom 404s. This is extremely hard to debug in production because the data is there — it's just on the "wrong" shard from the routing function's current perspective. This is why library selection and testing of the hashing implementation matters critically.
