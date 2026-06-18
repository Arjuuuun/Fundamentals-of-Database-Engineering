# Database Indexing — Complete Notes

> Source: Hussein Nasser — Fundamentals of Database Engineering
> Section: Indexing — What indexes are, how they work, when they fail, best practices

---

## Glossary — Terms Used in This Section

Before anything else, plain definitions of every term you'll see repeatedly:

| Term | Plain meaning |
|---|---|
| **Heap** | The actual table data stored on disk — all rows, all columns, no particular order. Think of it as the "raw storage" of your table. |
| **Page** | A fixed-size chunk of disk space (8KB in Postgres, 16KB in MySQL). The database always reads/writes in whole pages — never individual rows. One page typically holds several rows. |
| **IO** | One disk read/write operation. Always returns one or more whole pages. Fewer IOs = faster query. This is the main performance metric. |
| **B-tree** | The data structure most indexes use — a sorted tree where searching takes logarithmic time. Think of it like a sorted phone book with section tabs. |
| **Row ID / tuple ID (ctid)** | The internal address of a row — which page it's on and where within that page. Indexes store these to point back to the heap. |
| **Buffer pool** | An in-memory cache the database keeps for recently used pages. If a page is in the buffer pool, no disk IO needed — reads from RAM instead. LRU eviction (least recently used pages get dropped when full). |
| **Bitmap** | An array of bits (0s and 1s) where each bit represents one heap page. Bit=1 means "this page contains at least one matching row." Used to batch heap visits instead of random jumping. |
| **Selectivity** | How much an index narrows down results. High selectivity = index returns few rows (useful). Low selectivity = index returns most of the table (often useless). |
| **Cardinality** | Number of distinct values in a column. High cardinality (user_id, email) = good index candidate. Low cardinality (status: active/inactive, boolean) = often poor index candidate. |
| **Page split** | When a B-tree page is full and a new key must be inserted in the middle — the page splits into two. Expensive — causes IO, reference updates, potential cascading splits. |

---

## 1. What Is an Index?

> An index is a **separate data structure** built on top of a table that lets the database find rows without scanning the entire table.

Analogy: a phone book with alphabetical tabs. Instead of reading every name to find "Zebra Corp", you flip directly to Z. The index is that tab system.

**What an index contains:**
- The indexed column value(s)
- A pointer (row ID + page number) back to the actual row in the heap

**What an index does NOT contain:**
- All the other columns of the row (those live in the heap)

**The standard two-hop lookup:**
```
Step 1: Search the index (B-tree traversal) → find "employee_id=40 is on heap page 1, row 4"
Step 2: Jump to heap page 1 → fetch row 4 → return result
```

Two IOs total vs potentially hundreds for a full table scan.

**Index types:**
- **B-tree** — default in Postgres/MySQL. Works for equality (`=`) and range (`>`, `<`, `BETWEEN`) queries.
- **LSM tree** — used in write-heavy NoSQL systems. Different trade-offs.

---

## 2. Indexes Are Not Free

Every index has costs:

| Cost | Detail |
|---|---|
| **Write overhead** | Every INSERT/UPDATE/DELETE must update all indexes on that table. More indexes = slower writes. |
| **Storage** | Indexes are stored as pages on disk — they take up space. Large indexes may not fit in memory. |
| **Maintenance** | Postgres needs periodic `VACUUM` to clean up dead row versions that indexes still point to. |

**Rule:** only add indexes your actual query patterns need. Don't preemptively index everything.

---

## 3. The Three Scan Types

The database **always picks the cheapest strategy** based on estimated row count. Having an index doesn't guarantee it will be used.

### Sequential Scan (Full Table Scan)
> Read every page in the heap from start to finish. No index used.

```
Query: WHERE ssn = 666  (no index on ssn)
→ Read page 0: [rows 1,2,3] → check ssn → no match
→ Read page 1: [rows 4,5,6] → check ssn → no match
→ ... repeat for all 333 pages ...
→ Read page 332: [rows 998,999,1000] → MATCH
```

When Postgres chooses this:
- No index exists on the filtered column
- Query returns a large fraction of the table (index overhead > sequential scan cost)
- Statistics are stale and optimizer estimates wrong row count

### Index Scan
> Traverse B-tree → find row → **immediately jump to heap** for each match.

```
Query: WHERE id = 40  (index on id)
→ B-tree traversal → "id=40 is on heap page 1, row 4"
→ Jump to heap page 1 → fetch row 4 → done
```

Called **random access** because each heap jump lands on a different page. Gets expensive when many rows match — 1000 matches = 1000 random heap jumps.

When Postgres chooses this: small number of rows returned (high selectivity).

### Bitmap Index Scan (Postgres-specific, but concept exists elsewhere)
> Scan entire index first → build a bitmap of which heap pages contain matches → batch-fetch only those pages.

**Phase 1 — Bitmap Index Scan (index only):**
```
Scan the index for all matching rows
For each match: note which HEAP PAGE it lives on → set that bit in bitmap
Result: bitmap = [0,0,1,0,1,1,0,1,...]
                      ↑   ↑ ↑   ↑
                  page 2  4,5   7 contain matching rows
```

**Phase 2 — Bitmap Heap Scan:**
```
Look at bitmap → fetch ONLY flagged pages (batch, not random)
Recheck condition on each fetched row (page may contain non-matching rows too)
Return results
```

**Why recheck?** The bitmap tracks pages, not individual rows. A page might have 10 rows — maybe 7 match, 3 don't. Recheck filters the non-matching ones (cheap — page already in memory).

**The killer feature — combining multiple indexes:**
```sql
WHERE g > 95 AND id < 10000
-- g has an index, id has an index
```
```
Bitmap 1: scan g index → pages containing high-grade rows
Bitmap 2: scan id index → pages containing low-id rows
BitmapAND(bitmap1, bitmap2) → only pages satisfying BOTH conditions
→ dramatically fewer heap fetches than either index alone
```

When Postgres chooses this: moderate number of rows, or multiple indexes to combine.

### Decision Summary

```
Estimated rows returned:
  Very few  → Index Scan (direct heap jumps worth it)
  Moderate  → Bitmap Index Scan (batch heap access)
  Very many → Sequential Scan (just read everything, index overhead not worth it)
```

---

## 4. Index Only Scan — The Sweet Spot

> If ALL columns you SELECT are already IN the index — the database never visits the heap at all.

```sql
-- Index exists on (id)
SELECT id FROM employees WHERE id = 5000;
-- Index Only Scan: id is both the filter AND the select → heap never touched
-- Result: fastest possible query

SELECT name FROM employees WHERE id = 5000;
-- Index Scan: id found in index, but name is NOT in index → must visit heap
-- Result: one extra IO
```

**How to engineer Index Only Scans — the INCLUDE clause:**
```sql
-- You frequently run: SELECT id, name FROM employees WHERE id = ?
-- name is not in the index → always causes heap visit

CREATE INDEX idx_id ON employees(id) INCLUDE (name);
-- Now name is stored IN the index (as a non-key column)
-- Query becomes Index Only Scan → heap never visited
```

**Key distinction:**
- **Key columns** (in the main index definition) → used for searching/filtering/sorting
- **Non-key columns** (in INCLUDE) → stored in index for retrieval only, not searchable

**The trade-off:** larger index (stores extra columns) → may not fit in memory → could itself cause more IOs. Only use INCLUDE for genuinely frequent query patterns.

**VACUUM note:** Index Only Scans require the visibility map to be current. After bulk updates/deletes run `VACUUM table_name` to ensure `Heap Fetches = 0` in EXPLAIN output.

---

## 5. Reading EXPLAIN ANALYZE Output

> `EXPLAIN` shows the query plan (estimated). `EXPLAIN ANALYZE` runs the query and shows actual timings.

```sql
EXPLAIN ANALYZE SELECT name FROM grades WHERE id = 7;
```

```
Index Scan on grades_pkey  (cost=0.43..8.45 rows=1 width=19)
                                   ↑      ↑         ↑      ↑
                             startup  total    est.rows  row width(bytes)
  actual time=0.05..0.06 rows=1 loops=1
```

| Field | Meaning |
|---|---|
| Scan type | What strategy was used (Seq Scan, Index Scan, Index Only Scan, Bitmap...) |
| `cost=X..Y` | Estimated cost. X = startup (work before first row). Y = total. NOT milliseconds — relative units. |
| Startup cost | Work done BEFORE returning first row (sorting, hashing, etc.) |
| Total cost | Estimated work to return ALL rows |
| `rows` | Estimated rows returned |
| `width` | Average row width in bytes (affects network transfer cost) |
| `actual time` | Real time taken (only with ANALYZE) |

**Why startup cost matters:**
```
ORDER BY g  (g is indexed) → startup ≈ 0.43
  → index already sorted → can stream immediately

ORDER BY name  (name NOT indexed) → startup ≈ 1000+
  → must fetch ALL rows, sort everything, THEN return first row
  → cannot return anything until sort is complete
```

**Scan types in EXPLAIN output:**
- `Seq Scan` → full table scan ❌ (investigate why)
- `Index Scan` → index used, heap visited ✅
- `Index Only Scan` → index used, heap NOT visited ✅✅ (best)
- `Bitmap Index Scan` + `Bitmap Heap Scan` → batch index + heap ✅
- `Parallel Seq Scan` → multi-threaded full scan (Postgres spinning up workers)

**Read EXPLAIN bottom-up** — innermost operations happen first.

**width field — often overlooked:**
```
width=4   → INTEGER column (4 bytes)
width=19  → TEXT column (average)
width=31  → all three columns combined
```
Large width × millions of rows = significant network/memory cost. Never `SELECT *` across the wire when you only need one column.

---

## 6. When Indexes Get Ignored — Common Mistakes

Having an index doesn't mean it will be used. These patterns **silently bypass your index**:

```sql
-- ❌ Leading wildcard — B-tree has no starting point to search
WHERE name LIKE '%john%'
WHERE name LIKE '%john'

-- ✅ Trailing wildcard only — B-tree CAN search from prefix
WHERE name LIKE 'john%'

-- ❌ Function wrapping the column
WHERE UPPER(name) = 'JOHN'
WHERE DATE(created_at) = '2024-01-01'

-- ✅ Fix: rewrite without function on column
WHERE name = 'john'  -- if case-insensitive collation
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'

-- ❌ Arithmetic on column
WHERE salary + 100 > 50000

-- ✅ Fix: move arithmetic to the other side
WHERE salary > 49900

-- ❌ Implicit type cast
WHERE id = '1000'  -- id is INTEGER, '1000' is TEXT → cast forces seq scan
```

**Rule:** the column must appear alone on one side of the condition, without any transformation.

---

## 7. Composite Indexes — The Left-Most Column Rule

> A composite index `(a, b)` is a single B-tree sorted by `a` first, then `b` within each `a` group.

```sql
CREATE INDEX idx_ab ON test(a, b);
```

```
Index stores:
  (10, 5)   ← all a=10 entries grouped together
  (10, 80)
  (10, 100)
  (70, 20)  ← all a=70 entries grouped together
  (70, 80)
  (70, 100)
  (90, 5)
```

### 🔑 The Left-Most Column Rule (memorize this)

| Query | Uses idx_ab? | Why |
|---|---|---|
| `WHERE a = 70` | ✅ Yes | a is left-most → clear starting point in tree |
| `WHERE a = 70 AND b = 80` | ✅ Yes | Both columns → best case |
| `WHERE b = 80` | ❌ No | b alone → values scattered across entire tree, no starting point |
| `WHERE a = 70 OR b = 80` | ❌ Usually no | OR can't efficiently use composite |

**The b-alone problem visualized:**
```
WHERE b = 80:
  b=80 exists at (10,80), (70,80), (90,80) → scattered everywhere
  No position in tree to start searching for just "b=80"
  → full scan of index (or table) required
```

### Configurations — Which to Use

| Your query patterns | Best index config |
|---|---|
| Only `WHERE a` or `WHERE a AND b` | Composite `(a, b)` only |
| `WHERE a`, `WHERE b`, `WHERE a AND b` | Composite `(a, b)` + separate `(b)` |
| `WHERE a` and `WHERE b` independently, rarely together | Two separate indexes |
| High write volume | Minimize total indexes — each = write cost |

### OR Queries

```sql
WHERE a = 70 OR b = 80
```
- With composite only → seq scan (b-side has no individual index)
- With composite `(a,b)` + separate `(b)` → BitmapOr: use composite for a-side, separate index for b-side ✅

---

## 8. Query Optimizer — How It Chooses Between Indexes

The optimizer (query planner) runs on **every query** before execution — reads statistics, considers all possible plans, picks lowest estimated cost. You see its work in `EXPLAIN` output.

### Three Cases When Multiple Indexes Exist

**Case 1 — Use both:** moderate selectivity on both → BitmapAnd/BitmapOr both indexes, intersect/union row IDs, batch heap fetch.

**Case 2 — Use one, filter with the other:** one index far more selective → use the selective one, fetch matching rows from heap, apply second condition as in-memory filter. Common when one column is a primary key (always selective).

**Case 3 — Use neither:** both conditions return most of the table → seq scan cheaper than index traversal + random heap jumps.

### Table Statistics — The Optimizer's Brain

The optimizer's decisions are only as good as its statistics (row counts, value distributions, cardinality per column).

```sql
ANALYZE table_name;  -- refresh statistics in Postgres
-- Runs automatically via autovacuum, but not instantly after bulk operations
```

### 🔥 The Stale Statistics Trap (real production bug)

```
Fresh table: 3 rows → statistics say "tiny table"
Bulk INSERT: 300 million rows added
Query immediately after bulk insert:
  Optimizer reads stats → "this table has 3 rows, just seq scan"
  Actually scans 300 million rows → catastrophically slow
```

**Fix: always run `ANALYZE table_name` after bulk inserts before querying.**

### Index Selectivity — When Indexes Aren't Worth It

```sql
-- Table: 100M users, 95M from California
CREATE INDEX idx_state ON users(state);

WHERE state = 'California'  -- returns 95% of table → optimizer ignores index, seq scan
WHERE state = 'Florida'     -- returns 0.001% of table → index very useful
```

Low-cardinality columns (few distinct values: status, boolean, gender) often produce poor selectivity → index frequently ignored → wasted write overhead.

**Before creating an index, ask:** how many distinct values does this column have? How many rows will a typical query return?

---

## 9. CREATE INDEX CONCURRENTLY — Production Safety

Normal `CREATE INDEX` on a large table:
- **Blocks all writes** for the entire duration (minutes to hours)
- Reads still work
- Unacceptable in production

```sql
-- ❌ Blocks writes — never do this on live production table during business hours
CREATE INDEX idx_grade ON grades(g);

-- ✅ Non-blocking — safe for production
CREATE INDEX CONCURRENTLY idx_grade ON grades(g);
```

**How CONCURRENTLY works:**
- Does multiple passes over the table
- Pauses itself when write transactions are in progress
- Reads AND writes work throughout
- Takes longer (more CPU, more memory)
- Can fail → leaves index in invalid state

**The failure case:**
```sql
-- Building a unique index concurrently
-- During build, duplicates inserted (index didn't exist to block them)
-- → index ends up INVALID

-- Check for invalid indexes:
SELECT schemaname, tablename, indexname
FROM pg_indexes
JOIN pg_class ON pg_class.relname = indexname
WHERE pg_class.relkind = 'i'
AND NOT pg_index.indisvalid  -- requires joining pg_index too
-- simpler: just look for "INVALID" in \d table_name output in psql

-- Fix:
DROP INDEX CONCURRENTLY idx_grade;
CREATE INDEX CONCURRENTLY idx_grade ON grades(g);
```

**Rule:** always use `CONCURRENTLY` for production tables. Only skip during planned maintenance windows.

---

## 10. UUID v4 vs Sequential IDs — The Page Split Problem

### Why Indexes Must Stay Ordered

A B-tree index keeps all leaf page entries sorted. Every insert must land in the correct sorted position. If that position is on a **full page** → **page split**:

```
Leaf page: [10, 90]  ← full (max 2 entries)
Insert 80 → must go between 10 and 90
→ PAGE SPLIT:
  - Create new page
  - Redistribute entries: [10, 80] and [90]
  - Update parent node
  - Update linked list pointers between pages
  - Multiple IOs, multiple reference updates
```

### Random UUID v4 — Constant Page Splits

```
Insert UUID_a3f9... → lands on page 47
Insert UUID_02b1... → lands on page 203 (completely different location)
Insert UUID_f821... → lands on page 12
→ every insert touches a different page → constant page splits → IO thrashing
```

**Buffer pool thrashing:**
```
Insert UUID_A → fetch page 47 into buffer pool (RAM cache)
Insert UUID_B → fetch page 203 → evicts page 47 (LRU)
Insert UUID_C → fetch page 12 → evicts page 203
Insert UUID_D → needs page 47 again → fetch from disk again
→ endless cycle of fetch/evict/fetch → IO thrashing
```

### Sequential IDs — Clean Appends

```
Insert 10 → goes to last page
Insert 20 → goes to last page (or new rightmost page when full)
Insert 30 → goes to last page
→ always appending to the right → last few pages stay warm in buffer pool
→ no page splits until a page naturally fills up → new page at the end
```

### Real-World Example — Shopify

Shopify switched from UUID v4 to **ULID** (time-ordered, URL-safe) for purchase/order IDs:
- Purchases cluster in time → consecutive ULIDs sort together → inserts hit same index region
- Write performance improved significantly
- Read performance also improved — re-queries for recent purchases hit the same warm buffer pool region

**When sequential IDs help reads too:** only when your read pattern follows insert order (e.g., "show me recent purchases"). For random-access reads (URL shortener — any URL can be popular) — sequential IDs don't help reads.

### UUID Versions — Quick Reference

| Version | Ordered? | Safe? | Recommendation |
|---|---|---|---|
| v1 | ✅ Time-ordered | ❌ Leaks MAC address | Avoid |
| v4 | ❌ Random | ✅ | OK for non-indexed fields or low-write tables |
| v7 | ✅ Time-ordered | ✅ | Best modern choice for PKs |
| ULID | ✅ Time-ordered | ✅ | Popular alternative, URL-safe encoding |
| `BIGSERIAL` | ✅ Sequential | ✅ | Simplest choice for single-machine PKs |

**Practical rule:**
- High-write tables with indexed PK → `BIGSERIAL`, UUID v7, or ULID
- MySQL (clustered PK) → critical to avoid UUID v4 — the table itself must stay ordered
- Postgres (secondary PK) → less critical but still impacts index performance

---

## Working With Large Tables — Strategy Hierarchy

When a table grows to billions of rows:

```
1. Redesign schema     → avoid the large table entirely if possible
        ↓
2. Index               → reduce search space (B-tree narrows billions → thousands)
        ↓
3. Partition           → break table into smaller physical pieces on same machine
        ↓
4. Shard               → distribute partitions across multiple machines (adds complexity)
        ↓
5. Parallel brute force → MapReduce/parallel scan across cluster (batch/analytics only)
```

Always start at the top. Each step adds complexity — go only as far as your scale requires.

---

---

# ⚡ QUICK REVISION — Fast Interview Prep

## Scan Types — One Line Each

| Scan | When used | Heap visited? |
|---|---|---|
| Seq Scan | No index / returning most of table | Yes (entire table) |
| Index Scan | Index exists, few rows returned | Yes (per matching row) |
| Index Only Scan | All SELECT columns in index | ❌ No (fastest) |
| Bitmap Index Scan | Moderate rows / multiple indexes | Yes (batch, after bitmap built) |

## The Rules (Memorize These)

1. **Left-most column rule:** composite `(a,b)` index → only usable when query filters on `a`. Filtering on `b` alone = index ignored.

2. **Index killers:** `LIKE '%x%'`, functions on column (`UPPER(col)`), arithmetic on column (`col + 1`), implicit type cast → all bypass indexes silently.

3. **Index Only Scan:** achieved when all SELECTed columns are in the index. Use `INCLUDE` to add non-key columns.

4. **Stale statistics = wrong plan:** always `ANALYZE` after bulk inserts.

5. **CONCURRENTLY:** always use for production index creation. Can leave invalid index on failure — drop and recreate.

6. **UUID v4 on indexed columns = page splits:** use BIGSERIAL / UUID v7 / ULID for high-write indexed columns.

7. **Low cardinality = poor index:** indexing a boolean or status column with 2-3 values is often useless — optimizer will ignore it for most queries.

## EXPLAIN Output — What to Look For

```
Seq Scan       → no index used → investigate why (missing index? stale stats? low selectivity?)
Index Scan     → good, but check if heap fetches are avoidable (INCLUDE?)
Index Only Scan → best case
Bitmap Heap Scan + recheck → moderate case, multiple indexes combined
High startup cost → ORDER BY / sort on unindexed column → consider adding index
width is large → you're SELECT *-ing unnecessarily → select only needed columns
```

## Interview Pointers

**"Why would Postgres ignore an existing index?"**
Cost-based decision. If the query returns a large fraction of the table, seq scan is cheaper (one sequential pass vs index traversal + N random heap jumps). Also: stale statistics, expressions/functions on the column, low cardinality.

**"What is the left-most column rule?"**
A composite index `(a, b)` is sorted by `a` first. Querying on `b` alone means `b` values are scattered across the entire tree with no starting point — index unusable. Always filter on the left-most column.

**"How would you safely add an index to a production table?"**
`CREATE INDEX CONCURRENTLY` — non-blocking, allows reads and writes during build. Takes longer, uses more CPU/memory, can leave invalid index on failure (must drop and recreate). Never use plain `CREATE INDEX` on a live production table during business hours.

**"Why is UUID v4 bad for primary keys on high-write tables?"**
Random UUIDs insert at random positions in the B-tree → constant page splits → IO thrashing in the buffer pool (pages evicted and reloaded repeatedly). Sequential IDs always append to the rightmost page — no splits, buffer pool stays warm. Critical in MySQL (clustered PK = table must stay ordered). Use BIGSERIAL, UUID v7, or ULID instead.

**"What is an Index Only Scan and how do you achieve it?"**
When all columns in the SELECT are present in the index — the heap is never visited. Achieved by including frequently-selected columns in the index via `INCLUDE`. Check `Heap Fetches = 0` in EXPLAIN ANALYZE to confirm.

**"What's the difference between a bitmap index scan and a regular index scan?"**
Regular index scan: traverse B-tree → immediately jump to heap for each match (random access, expensive for many rows). Bitmap scan: traverse entire index first → build bitmap of matching heap pages → batch-fetch only those pages. Bitmap is better for moderate row counts and enables combining multiple indexes via BitmapAND/BitmapOR.

**"When would you NOT add an index?"**
High-write, low-read tables (every write must update the index). Low-cardinality columns (status, boolean — optimizer ignores it anyway). Very small tables (seq scan faster than index overhead). When the query returns most of the table regardless.

---

## Practical Commands Reference

```sql
-- Check query plan (estimated only, fast)
EXPLAIN SELECT ...;

-- Check query plan + actual execution (runs the query)
EXPLAIN ANALYZE SELECT ...;

-- Check plan + buffer/cache hits
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- Create index safely on production table
CREATE INDEX CONCURRENTLY idx_name ON table(column);

-- Refresh table statistics after bulk operations
ANALYZE table_name;

-- Clean up dead rows + update visibility map (enables Index Only Scan)
VACUUM table_name;
VACUUM VERBOSE table_name;  -- with details

-- Approximate row count (instant, good for display)
SELECT reltuples::BIGINT FROM pg_class WHERE relname = 'your_table';
-- use instead of SELECT COUNT(*) for large tables

-- Check for invalid indexes
SELECT indexname, tablename FROM pg_indexes
WHERE schemaname = 'public';
-- then \d table_name in psql to see INVALID marker
```

---

*End of Indexing section. Next section: [to be added]*
