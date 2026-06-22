# Section 5: Database Partitioning

> Source: Hussein Nasser — "Fundamentals of Database Engineering"
> Priority: 🔴 CORE | 📺 Must Watch (demo segment especially — EXPLAIN ANALYZE output with partition pruning)

---

## 1. What is Partitioning and Why Does it Exist?

**Partitioning** = splitting one large table into multiple smaller physical tables (called **partitions**), where the database engine automatically routes queries to the relevant partition(s) based on the `WHERE` clause — completely transparent to the client.

### The motivating problem

A table with 1M, 10M, or 1B rows eventually becomes slow and expensive to work with:
- **With an index**: B+ tree grows deeper as rows grow (Section 4), index maintenance gets heavier, buffer cache pressure increases
- **Without an index**: full table scan across millions of pages

> "The quickest way to query a table with a billion rows is to avoid querying a table with a billion rows."

That's the entire philosophy of partitioning — don't make the billion-row search faster; **avoid touching most of the billion rows entirely** by routing queries to a small, relevant slice first.

### How it works

1. The parent table becomes a **logical shell** — it holds no actual data, just metadata about partitions
2. Real rows live in partition tables, each covering a defined slice of data
3. On query, the engine checks partition metadata (cheap, no data scan) and **prunes** irrelevant partitions — this is called **partition pruning**
4. Only relevant partition(s) are touched; normal index/scan techniques then apply within that smaller set

**Example:** 1M-row `customers` table split into 5 partitions of 200K rows each, partitioned by `id` range. `WHERE id = 700001` → engine checks metadata → routes directly to the `customers_600001_800000` partition → works with 200K rows instead of 1M.

---

## 2. Horizontal vs. Vertical Partitioning

From this point on, **"partitioning" without a qualifier = horizontal partitioning.** Vertical partitioning is rarely used in practice today.

### Horizontal Partitioning (splitting by rows)

Think of a horizontal knife slicing through the table — each slice is a **subset of rows**, all with the **same columns**, living in its own physical partition table.

```
Original table (1M rows, columns: id, name, email, grade)
         ↓ split by id range
Partition 1: rows 1–200K     (same columns: id, name, email, grade)
Partition 2: rows 200K–400K  (same columns: id, name, email, grade)
Partition 3: rows 400K–600K  ...
```

### Vertical Partitioning (splitting by columns)

Think of a vertical knife — you're separating **certain columns** into their own physical table, while keeping all rows.

**Canonical use case:** a table has a `BLOB` column (Binary Large Object — a data type for storing large, unstructured data like documents, images, or large JSON payloads) that is:
- Rarely queried
- Extremely wide (takes large space per row)

**Without vertical partitioning:** every page read drags the BLOB along, even for queries that only need `name` and `email`. Wide rows → fewer rows per page → more IOs → slower everything. Ties directly to Section 2 (storage, row width, pages) and Section 4 (B+ tree fan-out is hurt by wide keys).

**With vertical partitioning:** BLOB column moved to a separate table on cheaper/slower storage; main table becomes compact and fast. Engine knows to join into the BLOB table only when that column is actually needed — transparent to the query.

**Relationship to normalization:** vertical partitioning is functionally similar to normalization (splitting wide tables into narrower related tables). Normalization is a design-time principle; vertical partitioning is an operational/performance technique applied after the fact. This is why vertical partitioning is rarely needed — good schema design handles it upfront.

---

## 3. Partitioning Types

### Range Partitioning
Rows assigned based on a **value falling within a defined range** of the partition key.

```sql
PARTITION BY RANGE (g)
-- or
PARTITION BY RANGE (log_date)
```

**Examples:**
- IDs 1–200K → partition 1, 200K–400K → partition 2
- IoT measurements: 2022 data → one partition, 2023 → next, 2024 → next

**Unique advantage:** old partitions can be **detached and moved to slower/cheaper storage** (tape archive, cold object storage) without touching the rest of the table — **tiered storage**. Nobody queries 1995 data; why keep it on expensive NVMe SSD?

**Watch out for:** uneven data distribution across ranges creating "hot" partitions.

---

### List Partitioning
Rows assigned based on **matching a specific discrete value** — not a range.

```sql
PARTITION BY LIST (state)
-- California customers → partition_ca
-- Alabama customers   → partition_al
```

**Examples:** countries, states, zip codes, categories, regions.

**Range vs. List:** range = "between X and Y" (continuous values); list = "equals this exact value" (categorical/discrete values).

**Watch out for:** same hotspot risk — if California has 10x more customers than Wyoming, `partition_ca` becomes huge.

---

### Hash Partitioning
A **hash function** applied to the partition key deterministically assigns each row to a partition.

```
hash(customer_id) % 4 → values 0, 1, 2, 3 → four partitions
```

**Examples:**
- Cassandra uses hash partitioning as its core data distribution mechanism
- Load balancers use "IP hash" to route requests — same client IP always hits the same backend

**Why it exists:** range and list partitioning can create uneven partition sizes. Hash partitioning **spreads rows evenly** regardless of actual values — the hash function distributes uniformly by design.

**Critical tradeoff:** hash partitioning **kills range query pruning.** If partitioned by `id` hash, `WHERE id BETWEEN 100 AND 200` can't prune any partitions — the engine has no idea which partitions those IDs landed in (hashing scrambles ordering). Optimized for **exact single-row lookups**, not range scans.

---

### Comparison

| Type | Assigned by | Best for | Watch out for |
|---|---|---|---|
| **Range** | Value within a defined range | Dates, IDs, continuous ordered data; tiered storage | Hot partitions if data skewed |
| **List** | Value matches a discrete value | Categorical data (countries, states, categories) | Same hotspot risk |
| **Hash** | Hash function output | Even distribution, exact single-row lookups | Kills range query pruning |

---

## 4. Partitioning vs. Sharding

A critical distinction — asked constantly in senior interviews.

### Partitioning
- Splits table into multiple physical tables → **same database, same server**
- DB engine manages all routing — **client is completely agnostic** (sends normal query to parent table, engine decides which partition to hit)
- Physical table names change under the hood (`customers_0_200k`, etc.) but client never sees this — just queries `customers`
- Complexity lives **inside the database engine**

### Sharding
- Also splits table into multiple physical tables → but on **completely different database servers**
- Done for distributed scale and geographic latency (e.g., California customers on US-West server, Asian customers on APAC server)
- **Client/proxy layer must be aware** of which server to target — routing logic (IP detection, user region, customer ID range) lives outside the DB
- Table name stays the same across shards — the query doesn't change, but the **server it executes against changes**
- Technologies like **Vitess** and **ProxySQL** attempt to abstract this, making sharding look more like partitioning from the client's perspective — but it's still a heavy architectural investment
- Client awareness introduces serious distributed systems problems: cross-shard queries, rebalancing when shards grow, distributed transactions

### Comparison

| | Partitioning | Sharding |
|---|---|---|
| Where do splits live? | Same DB, same server | Different servers entirely |
| Who manages routing? | DB engine — transparent | Client/proxy — application must know |
| Client awareness | Zero — fully agnostic | Yes — knows which shard to target |
| Table name | Changes per partition (hidden) | Same table name, different server |
| Main motivation | Query performance on large tables | Distributed scale, geographic latency |
| Complexity | Low-moderate (DB manages it) | High (distributed systems problems) |
| Example tech | Postgres native partitioning | Vitess, ProxySQL, app-level routing |

**One-line interview answer:** "Partitioning splits a table into multiple physical tables on the **same server** — routing is fully transparent to the client. Sharding also splits the table but across **different servers** — the client or a proxy must be aware of which server to target, introducing significant distributed-systems complexity."

---

## 5. Postgres Partitioning — Practical Setup

### Step 1: Create the parent table

```sql
CREATE TABLE grades_parts (
    id SERIAL NOT NULL,
    g  INTEGER NOT NULL
) PARTITION BY RANGE (g);
```

- `PARTITION BY RANGE (g)` — declares this as a partitioned parent table, partitioned by range on `g`
- This table **holds no actual data** — it's a routing shell
- Partition key column **must be NOT NULL** — nullable partition keys make routing ambiguous

### Step 2: Create partition tables

```sql
CREATE TABLE g0035  (LIKE grades_parts INCLUDING INDEXES);
CREATE TABLE g3560  (LIKE grades_parts INCLUDING INDEXES);
CREATE TABLE g6080  (LIKE grades_parts INCLUDING INDEXES);
CREATE TABLE g80100 (LIKE grades_parts INCLUDING INDEXES);
```

- `LIKE grades_parts INCLUDING INDEXES` — copies exact column structure and any indexes from the parent
- These are just normal standalone tables at this point — not yet wired to the parent

### Step 3: Attach partitions with range boundaries

```sql
ALTER TABLE grades_parts ATTACH PARTITION g0035  FOR VALUES FROM (0)  TO (35);
ALTER TABLE grades_parts ATTACH PARTITION g3560  FOR VALUES FROM (35) TO (60);
ALTER TABLE grades_parts ATTACH PARTITION g6080  FOR VALUES FROM (60) TO (80);
ALTER TABLE grades_parts ATTACH PARTITION g80100 FOR VALUES FROM (80) TO (100);
```

**Critical boundary rule:** Postgres ranges are **inclusive on lower bound, exclusive on upper bound.** `FROM (35) TO (60)` means `g >= 35 AND g < 60`. This is why ranges chain cleanly without overlap — 35 goes to `g3560`, not `g0035`.

### Step 4: Insert data (always target the parent)

```sql
INSERT INTO grades_parts (g) VALUES (42);  -- engine routes to g3560 automatically
```

Never insert directly into a partition table in normal operation — always use the parent table. The engine routes each row to the correct partition based on the value.

### Resulting structure

```
grades_parts (parent — no data, just routes queries)
├── g0035   (grades 0–34)
├── g3560   (grades 35–59)
├── g6080   (grades 60–79)
└── g80100  (grades 80–99)
```

---

## 6. Automating Partition Creation

Manual creation doesn't scale — 100 partitions means 200 SQL statements. Several approaches to automate:

### Option 1: Application-level script (Node.js / any language)

```javascript
const client = new Client({ /* postgres connection */ });

for (let i = 0; i < 100; i++) {
    const idFrom = i * 10_000_000;
    const idTo   = (i + 1) * 10_000_000; // exclusive upper bound

    const partitionName = `customers_${idFrom}_${idTo}`;

    await client.query(`
        CREATE TABLE ${partitionName} 
        (LIKE customers INCLUDING INDEXES)
    `);

    await client.query(`
        ALTER TABLE customers 
        ATTACH PARTITION ${partitionName} 
        FOR VALUES FROM (${idFrom}) TO (${idTo})
    `);
}
```

**Gotcha:** `TO` value is exclusive — so `FROM (0) TO (10000000)` covers IDs 0–9,999,999. Getting this wrong leaves gaps between partitions where inserts will fail.

### Option 2: PL/pgSQL (native Postgres — no external dependency)

```sql
DO $$
DECLARE
    i       INTEGER;
    id_from BIGINT;
    id_to   BIGINT;
BEGIN
    FOR i IN 0..99 LOOP
        id_from := i * 10000000;
        id_to   := (i + 1) * 10000000;
        EXECUTE format(
            'CREATE TABLE customers_%s_%s (LIKE customers INCLUDING INDEXES)',
            id_from, id_to
        );
        EXECUTE format(
            'ALTER TABLE customers ATTACH PARTITION customers_%s_%s
             FOR VALUES FROM (%s) TO (%s)',
            id_from, id_to, id_from, id_to
        );
    END LOOP;
END $$;
```

Good for one-time setup. No external dependency — runs entirely inside Postgres.

### Option 3: Migration tools (Flyway, Liquibase, raw .sql files)

Generate partition SQL once via a script, commit the output as a versioned migration file, run it through your migration tool. Most production-safe approach — version-controlled, reviewable, repeatable.

### Option 4: pg_partman (Postgres extension)

A dedicated Postgres extension for partition management — handles automatic partition creation, maintenance schedules, and retention policies (e.g., auto-create next month's time-series partition, auto-drop partitions older than 2 years). The most robust production solution for time-series/append-heavy workloads.

---

## 7. Performance Demo Results (Postgres, 10M rows)

Baseline: `grades_original` — single unpartitioned table, 10M rows, index on `g`.

| Query | Unpartitioned | With partitioning | Scan type used |
|---|---|---|---|
| `WHERE g = 30` | ~2 seconds | Hits 1 partition only | Bitmap Index Scan |
| `WHERE g BETWEEN 30 AND 35` | ~3 seconds | Hits 1–2 partitions | Parallel Index Scan |

`EXPLAIN ANALYZE` reveals which partitions were hit — this is how you verify partition pruning is actually working for your query patterns in practice. If all partitions appear in the plan, your query isn't benefiting from pruning.

---

## 8. Pros and Cons

### Advantages

**1. Improved query performance via partition pruning**
Engine skips irrelevant partitions entirely. A query on a 10M-row table partitioned into 4 ranges might only scan 2.5M rows instead of all 10M.

**2. Smaller, faster indexes per partition**
Instead of one massive B+ tree spanning all 10M rows, each partition has its own smaller, shallower B+ tree. Connects to Section 4: smaller tree = fewer levels = fewer IOs = easier to cache in RAM (buffer cache).
*Example:* index on `g` over 10M rows vs. four separate indexes each covering ~2.5M rows — each fits more comfortably in Postgres's buffer cache.

**3. Tiered storage for old data**
Old, rarely-queried partitions (e.g., 2019 IoT data) can be detached and moved to cheaper/slower storage without touching the rest of the table. Impossible to do cleanly with a monolithic table.

**4. Faster bulk operations**
`DELETE`, `VACUUM`, `REINDEX` only touch the target partition's pages. Dropping old time-series data? `DROP TABLE measurements_2020` or `DETACH PARTITION` — instant, no vacuum needed. Much faster than `DELETE FROM measurements WHERE year = 2020` on a monolithic table.

**5. Better parallelism**
Multiple partitions can be scanned in parallel by different workers — more natural parallel execution than splitting a single huge table.

### Disadvantages

**1. Partition key queries only — wrong column kills the benefit**
Queries filtering on a column other than the partition key can't prune — engine scans all partitions. Potentially as slow or slower than the original unpartitioned table plus overhead.
*Example:* table partitioned by `g` (grade), query `WHERE id = 700001` → scans all four partitions, zero benefit.

**2. Manual partition management — no auto-splitting**
Postgres doesn't auto-create new partitions as data grows. Insert a value outside all defined ranges → insert fails. Must pre-create partitions or automate creation (pg_partman, scripts).
*Example:* time-series table partitioned by month — forget to pre-create next month's partition → inserts start failing in production at midnight on the 1st.

**3. Cross-partition queries are expensive**
Queries without a partition key filter must touch every partition and merge results — sometimes worse than an unpartitioned table due to coordination overhead.
*Example:* `SELECT AVG(g) FROM grades_parts` — no WHERE on `g`, engine hits all 4 partitions, computes partial aggregates, merges.

**4. Partition key must be NOT NULL**
Postgres requires the partition key column to be `NOT NULL`. Can force schema changes on existing tables.

**5. Hotspot risk with range/list partitioning**
Uneven data distribution makes some partitions huge, others tiny — defeats the purpose of partitioning.
*Example:* grades table where 80% of students score 60–80. `g6080` becomes enormous while others are near-empty. Rebalancing partition boundaries on a live table is expensive.

---

## Glossary

| Term | Definition |
|---|---|
| **Partitioning** | Splitting a large table into multiple smaller physical tables on the same server, with DB-managed routing. |
| **Partition** | One physical slice of a partitioned table — a normal table under the hood. |
| **Parent table** | The logical shell/router — holds no data, just metadata about partitions. |
| **Partition pruning** | The DB engine eliminating irrelevant partitions before touching any data, based on WHERE clause and partition metadata. |
| **Horizontal partitioning** | Splitting by rows — each partition has the same columns, different row ranges. The standard meaning of "partitioning." |
| **Vertical partitioning** | Splitting by columns — separating wide/rarely-used columns into a separate table. Similar to normalization. |
| **Range partitioning** | Rows assigned based on value falling within a defined range (dates, IDs, etc.). |
| **List partitioning** | Rows assigned based on matching a discrete value (countries, states, categories). |
| **Hash partitioning** | Rows assigned based on hash function output — ensures even distribution but kills range query pruning. |
| **Sharding** | Splitting a table across different database servers — client/proxy must be aware of routing. More complex than partitioning. |
| **Tiered storage** | Storing data on different storage media based on access frequency — hot recent data on fast SSD, cold old data on cheap/slow storage. |
| **BLOB** | Binary Large Object — data type for storing large unstructured data (documents, images, large JSON). Common trigger for vertical partitioning. |
| **pg_partman** | Postgres extension for automated partition management and retention policies. |
| **Partition key** | The column used to determine which partition a row belongs to. Must be NOT NULL in Postgres. |

---

## Quick Revision (skim before an interview)

1. Partitioning = splitting one big table into smaller physical tables on the **same server** — DB routes queries automatically, client is fully agnostic.
2. Horizontal = split by rows (same columns, different row ranges). Vertical = split by columns (rarely used, similar to normalization).
3. Three types: **Range** (continuous values, good for dates/IDs, enables tiered storage), **List** (discrete values, categorical data), **Hash** (even distribution, kills range pruning).
4. Parent table = empty shell/router. Real data lives in partition tables. Insert only to parent — engine routes to correct partition.
5. Partition key column must be NOT NULL. Postgres ranges are lower-inclusive, upper-exclusive.
6. Partition pruning = engine skips irrelevant partitions based on WHERE clause. Only works if you filter on the partition key.
7. Wrong-column queries = no pruning = scan all partitions = potentially worse than unpartitioned.
8. Partitioning ≠ sharding. Partitioning: same server, DB manages routing. Sharding: different servers, client/proxy must know where to route.
9. Old partitions can be detached and moved to cold storage — impossible cleanly with monolithic tables.
10. Automate partition creation at scale: PL/pgSQL, application scripts, migration files, or pg_partman extension.

---

## Interview Questions & Answers

**Q1: What is database partitioning and why would you use it?**
A: Partitioning splits a large table into multiple smaller physical tables (partitions) on the same database server. The DB engine automatically routes queries to the relevant partition(s) based on the WHERE clause — completely transparent to the client. You use it when a table becomes too large for efficient querying, indexing, or maintenance — the goal is to let the engine skip irrelevant data entirely (partition pruning) rather than scanning the full table, even with indexes.

**Q2: What is partition pruning and how does it work?**
A: Partition pruning is when the database engine eliminates irrelevant partitions before touching any actual data. The engine checks partition metadata (a fast, cheap operation — not a data scan) against the WHERE clause, identifies which partition(s) could contain matching rows, and queries only those. A query with `WHERE id = 700001` on a table partitioned by ID ranges instantly routes to the single relevant partition, skipping all others entirely.

**Q3: What's the difference between horizontal and vertical partitioning?**
A: Horizontal partitioning splits by rows — each partition has the same column structure but a different row range (e.g., IDs 0–200K, 200K–400K). This is what people mean by "partitioning" in practice. Vertical partitioning splits by columns — separating wide or rarely-accessed columns (like BLOB fields) into a separate table to make the main table more compact and faster to read. Vertical partitioning is rarely needed today since good schema normalization handles it at design time.

**Q4: What are the three main partitioning types and when would you use each?**
A: Range partitioning assigns rows based on a value falling within a defined range — best for dates, sequential IDs, and any continuously ordered data. It also enables tiered storage (detach old partitions to cold storage). List partitioning assigns rows based on matching a discrete value — best for categorical data like countries or states. Hash partitioning applies a hash function to spread rows evenly regardless of their values — best when you need even distribution and only do exact single-row lookups, since it destroys range query pruning.

**Q5: What is the difference between partitioning and sharding?**
A: Partitioning splits a table into multiple physical tables on the same database server — routing is fully managed by the DB engine, the client is completely agnostic. Sharding also splits the table but across different database servers — the client or a proxy layer must be aware of which server holds the relevant data. Sharding introduces significant distributed-systems complexity: cross-shard queries, rebalancing, distributed transactions. Partitioning keeps all that complexity inside the engine where it belongs.

**Q6: What happens if you query a partitioned table on a column that isn't the partition key?**
A: Partition pruning can't happen — the engine has no metadata to determine which partition(s) contain matching rows for a non-partition-key column, so it scans all partitions. This can be as slow or slower than the original unpartitioned table, plus the overhead of coordinating multiple tables. This is why choosing the right partition key — one that aligns with your actual query patterns — is the most critical partitioning design decision.

**Q7: What are the main disadvantages of partitioning?**
A: The key ones: (1) Queries not filtering on the partition key get no pruning benefit — potentially slower than unpartitioned. (2) Postgres doesn't auto-create partitions — you must pre-create them manually or via automation; missing a partition means inserts fail. (3) Cross-partition aggregations (no WHERE on partition key) have merge overhead. (4) Partition key must be NOT NULL. (5) Range/list partitioning can create hotspot partitions if data isn't evenly distributed.

**Q8: Why does partitioning improve index performance, not just query routing?**
A: Each partition gets its own independent B+ tree index covering only its subset of rows. A smaller table means a shallower B+ tree (fewer levels, fewer IOs per lookup from Section 4's fan-out math) and a smaller index that fits more easily in Postgres's buffer cache (RAM). Instead of one massive deep index over 10M rows, you get four shallower indexes each over 2.5M rows — each faster to traverse and easier to keep hot in memory.

**Q9: How does tiered storage work with partitioning, and why can't you do this with a monolithic table?**
A: With range partitioning on a time-based column, old partitions (e.g., 2019 IoT data) can be detached from the parent table and moved to cheaper/slower storage (cold HDD, tape archive, object storage), then re-attached. Queries for recent data are completely unaffected. With a monolithic table, all rows live in the same set of data files — you can't physically separate old data onto different storage without restructuring the entire table.

**Q10: What is the Postgres range boundary rule, and why does it matter?**
A: Postgres partition ranges are lower-bound inclusive, upper-bound exclusive. `FOR VALUES FROM (35) TO (60)` means `g >= 35 AND g < 60`. This matters because getting boundaries wrong leaves gaps between partitions where inserts will fail with a "no partition found" error. When automating partition creation, the upper bound variable must be the next partition's start value (not current partition's end value minus 1), otherwise you silently leave a one-value gap.

**Q11: When would you choose hash partitioning over range partitioning?**
A: Hash partitioning when your primary concern is even data distribution and your main access pattern is exact single-row lookups (e.g., `WHERE customer_id = 12345`). Range partitioning when your queries frequently filter on ordered ranges (dates, ID ranges) or when you need tiered storage for old data. Hash partitioning is a poor choice if you have range queries (`BETWEEN`, `<`, `>`) since hashing scrambles the ordering and makes range pruning impossible.

**Q12: How would you handle a partitioned table that needs a new partition added as data grows?**
A: Several approaches: manually run `CREATE TABLE ... (LIKE parent INCLUDING INDEXES)` + `ALTER TABLE parent ATTACH PARTITION ... FOR VALUES FROM ... TO ...` before the new range's data arrives; automate via a scheduled application script or PL/pgSQL procedure; or use the `pg_partman` extension, which handles automatic partition creation and retention policies natively. The critical operational requirement is that the new partition must exist *before* the first row in its range is inserted — missing it causes insert failures.
