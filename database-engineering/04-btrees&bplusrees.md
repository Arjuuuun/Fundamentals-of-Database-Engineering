# Section 4: B-Trees and B+ Trees

> Source: Hussein Nasser — "Fundamentals of Database Engineering"
> Priority: 🔴 CORE | 📺 Must Watch (entire section — visual traversal is hard to follow without diagrams)

---

## 1. The Problem: Full Table Scans

A **full table scan** is the slowest way to find a row — the engine reads every row in the table, with no shortcuts.

Mechanically, a table is one or more data files on disk. The engine doesn't read "rows" directly — it reads **pages** (fixed-size chunks):

- Postgres: **8 KB** pages by default (page size requires recompiling from source to change)
- MySQL: **16 KB** pages by default

Each page-read ≈ one **IO** (input/output operation — one request to disk for a chunk of data). Underneath the database's logical page, physical disk hardware has its own block/sector size (e.g., SSD physical page ≈ 4 KB) — there's a translation layer here, but it's implementation detail, not something you reason about day-to-day.

**Cost of a full scan = number of pages touched, not number of rows.** A table with many small rows packed tightly can scan faster than one with fewer, wider rows — because fewer pages are touched.

**Important nuance:** "Worst case = target row is last" doesn't map cleanly onto real engines. Production DBMSs aren't naive sequential readers — they use parallel workers, scan from both ends inward, and split work across threads even during a full scan.

**The motivating problem:** we need a way to reduce the search space — to jump directly to relevant pages instead of touching all of them. B-trees (and B+ trees) are the answer.

---

## 2. B-Tree Fundamentals

**B-tree** = a balanced tree data structure designed to minimize the number of IOs needed to find a row.

The name "B" is ambiguous even in the original 1970 paper — could stand for **Balanced**, **Boeing** (the research lab it came from), or **Bayer** (Rudolf Bayer, one of the original authors). ⚪ trivia only.

### Core vocabulary

| Term | Definition |
|---|---|
| **Node** | A unit of the tree. In real databases: **one node = one disk page** (not an abstract textbook box). |
| **Degree (M)** | Max number of child nodes a node can have. Chosen automatically by the engine based on page size + key width — not a tunable setting for developers. |
| **Elements per node** | Always `M − 1`. (5 children → 4 elements.) |
| **Element** | A key-value pair. |
| **Key** | The thing you're searching for (e.g., an indexed column's value). |
| **Value / Data pointer / α (alpha)** | Pointer to the actual row. Target differs by database: **Postgres** secondary indexes point directly to the **tuple**; **MySQL/InnoDB and Oracle** secondary indexes point to the **primary key** (requiring an extra lookup step). This is part of why Uber famously migrated from Postgres to MySQL. |
| **Tuple ID / Row ID** | The internal, physical location identifier for a row — distinct from your logical primary key column. Sometimes encodes the page number directly. |
| **Root node** | Top of the tree, single entry point. |
| **Internal node** | Middle layers, used for navigation/routing only. |
| **Leaf node** | Bottom layer, no children. |
| **Child pointer** | A separate pointer (distinct from the data pointer) used purely to navigate from a node to its child during traversal. |

### Traversal rule

For a node with elements `[2, 4]`:
- Left child → keys `< 2`
- Middle child → keys between `2` and `4`
- Right child → keys `> 4`
- **Equal-key tie-breaker**: if your search key exactly equals a routing key, go **right**.

### Why "thousands of elements per node" matters — the fan-out chain

This is the central causal chain of the entire section:

```
Node = disk page (fixed size, e.g. 8 KB)
   ↓
Key width determines how many elements fit per page → this is the FAN-OUT (M)
   ↓
High fan-out → tree needs fewer LEVELS to hold the same number of rows
   ↓
Fewer levels = fewer IOs per search (each level traversed = 1 IO)
```

**Worked example (Postgres, 8-byte BIGINT key + ~6-byte tuple pointer ≈ 16 bytes/element):**

```
8192 bytes / 16 bytes per element ≈ 512 elements per node (fan-out ≈ 500)
```

| Level | Nodes at level | Rows reachable |
|---|---|---|
| Root | 1 | 500 |
| Level 2 | 500 | 250,000 |
| Level 3 | 250,000 | 125,000,000 |

**A 100M-row table fits in a 3–4 level tree → 3–4 IOs to find any row.** A toy university example (M=3) would need ~17 levels for the same row count — 17 IOs vs. 4.

**Key insight:** index depth is a property of **each individual index**, not the table — driven by:
1. **Key width** (BIGINT vs UUID vs VARCHAR — narrower = more fan-out)
2. **Composite index width** (multiple columns = wider key = somewhat lower fan-out)
3. **Row count** (more rows needs more levels, but fan-out's 500x multiplier per level means row count has to grow enormously before depth increases by even one level)
4. **Page size** (fixed per database engine: 8 KB Postgres, 16 KB MySQL — not per-table)

Every index on a table gets its **own separate B+ tree**. A table with no indexes is just an unordered heap — full scans only.

### B-tree search cost is inconsistent

Because keys+values can live at **any level** (root, internal, or leaf) depending on where they landed during insertion:
- Best case: key found at root → **1 IO**
- Worst case: key found at a leaf → full tree depth in IOs

This inconsistency is itself one of the limitations B+ trees fix (see below).

### Insertion / splitting (🟡 condensed)

- Nodes fill up to their element limit (`M − 1`); inserting beyond that triggers a **split** — node breaks in two, a key gets pushed to the parent.
- Splits are costly — this is *why* nodes are deliberately sized to a full page (maximize capacity) rather than small.
- **Inserting in random order is slower than sequential order** — random keys scatter across pages instead of filling them in order, triggering more splits. This is the structural root cause of **the UUID v4 problem** from Section 3 — random primary keys cause more page splits/fragmentation on insert than sequential (auto-increment) keys.

---

## 3. B-Tree Limitations (why B+ trees exist)

### Limitation 1 — Every node stores both keys AND values → wasted space

Every node, including internal/root nodes used purely for *navigation*, carries the full value (data pointer) — even though that value is never used until you've actually found your target leaf. This wastes space at every level you pass through on the way down.

This compounds badly when the **key itself** is wide (UUID = 16 bytes, string/VARCHAR = variable, possibly large) — fewer elements fit per page → lower fan-out → deeper tree → more IOs. (Note: "value" here specifically = the data pointer, typically small/fixed-size; the bigger practical hit usually comes from wide **keys**, a related but distinct cost.)

### Limitation 2 — Range queries are slow (random access / "thrashing")

Even though keys like `4, 5, 6, 7, 8, 9` are logically adjacent and sorted, in a plain B-tree they can each land on **completely different nodes** depending on insertion history. Fetching a range means separately searching for each value — jumping root → leaf → internal → different branch — repeatedly. This is called **thrashing the index**.

This matters enormously in practice: range queries (`BETWEEN`, `<`, `>`, `ORDER BY ... LIMIT`) are far more common in real applications than single-row lookups.

### Limitation 3 — Hard to fit internal nodes in memory

Direct consequence of Limitation 1: if every node (including pure-navigation internal nodes) carries full key+value pairs, and values/keys are wide, the **entire tree becomes expensive to cache in RAM** (buffer cache) — working against the database's goal of avoiding disk IO altogether.

---

## 4. B+ Trees — The Production Standard

**B+ tree = B-tree, but values are relocated to leaves only.**

| | B-tree | B+ tree |
|---|---|---|
| Key location | Exactly once, anywhere in the tree | **Duplicated** — routing copy in internal/root nodes (key only, no value), real entry at leaf |
| Value location | Wherever the key happens to be | **Only at leaf level**, alongside the real key copy |
| Search cost | Variable (1 IO if lucky / found at root, up to full depth if not) | **Constant** — always full tree depth, since real data only exists at leaves |
| Internal node contents | Keys + values | **Keys only** (slim) |
| Range query performance | Poor — random access / thrashing | Strong — sequential, via linked leaves |

### Structure

- **Root + internal nodes**: keys only, no values. Pure routing/navigation — "go left/middle/right."
- **Leaf nodes**: every key that exists in the index, with its value (data pointer), lives here — **and only here**.
- **Key duplication**: a key like `7` may exist twice — once as a bare routing signpost in an internal node (no value), once as the real entry (with value) at the leaf. Not every key is duplicated (some never get chosen as routing signposts), but it's a structural norm. The storage cost of this duplication is proven to be minimal relative to the fan-out gains.
- **Leaf linking**: leaf nodes are typically linked to each other in sorted order (each leaf points to the next). **Not universally implemented across every database**, but a common B+ tree feature. This is what fixes the range-query problem — walk sideways along leaves instead of re-traversing the tree.

### Why every search must reach a leaf

Because **all real data (values) exists only at the leaf level**, every search — no matter the key — must traverse all the way down, even if a matching routing key appears in the root. That root copy has no value attached; it's structurally useless for retrieval. Root/internal keys exist only to route you, never to answer the query directly.

**Tradeoff:**
- Downside: every search costs the *maximum* possible IOs (full depth) — no "lucky shortcut" case like plain B-trees have.
- Upside: predictable, constant-cost search.
- Bigger upside: because internal nodes are slimmer (keys only), fan-out is much higher than an equivalent B-tree holding the same data — so "all the way down" is a *shorter* trip than it would've been otherwise. Net effect: B+ trees usually win on total IO count despite the seemingly strict requirement.

### Range query example (the payoff)

Find all rows with ID between 4 and 9:
1. Traverse root/internal nodes to locate the starting leaf containing `4` (a couple of IOs).
2. Once on that leaf, `4, 5, 6, 7, 8, 9` sit **contiguously, in sorted order**, on the same leaf page (or a couple of linked leaf pages).
3. **Best case: the entire range query costs as little as 1 extra IO** — versus a plain B-tree needing several separate, scattered traversals to collect the same six values.

This is the direct fix for Limitation 2: range queries go from "many random-access traversals" to "one sequential walk along linked, sorted leaves."

### Realistic scale (not toy diagrams)

Diagrams typically show ~5 elements per node only because that's drawable. Real pages hold **400–500+ elements**, consistent with the fan-out math in Section 2. A 4-byte integer key packs hundreds of elements per page; a UUID, long string, or blob packs far fewer — directly hurting both fan-out *and* range-query density (fewer relevant rows per leaf page = more IOs for the same range).

---

## 5. Production Database Reality Check

- **Every mainstream relational database's default index is a B+ tree** — Postgres, MySQL/InnoDB, Oracle, SQL Server. What `CREATE INDEX` builds by default is a B+ tree.
- **Naming mismatch**: all of them call it a "B-tree" in docs and commands (Postgres index type: `btree`; MySQL: `BTREE`) even though what's actually implemented is a B+ tree. No production system uses the pure, original B-tree.
- **Not every index type is a B+ tree, even within one database.** Postgres also offers hash indexes, GIN/GiST (full-text search, JSON, geometric data), BRIN (very large, naturally-ordered tables). B+ tree (`btree`) is the *default*, not the *only* option.
- **Different storage architectures use different structures entirely.** Write-heavy-optimized engines (Cassandra, RocksDB-based systems like MySQL's MyRocks) use **LSM trees** (Log-Structured Merge trees) instead of B+ trees — a different structure optimized for write throughput rather than read/range-query performance. MongoDB's default engine (WiredTiger) does use B+ trees.

---

## Glossary

| Term | Definition |
|---|---|
| **Full table scan** | Reading every row/page in a table sequentially with no index shortcut. |
| **Page** | Fixed-size unit of disk storage the DB reads/writes in (8 KB Postgres, 16 KB MySQL default). |
| **IO** | One input/output operation — roughly one page read from disk. |
| **Node** | A B-tree/B+ tree unit; in production, equals one disk page. |
| **Degree (M)** | Max children a node can have; determines fan-out. |
| **Fan-out** | Number of children/elements a node can hold — directly controls tree depth. |
| **Key** | The searchable value in an index element (e.g., an ID). |
| **Value / Data pointer (α)** | Pointer from an index entry to the actual row. |
| **Tuple ID / Row ID** | Internal physical row locator, distinct from the logical primary key. |
| **Root node** | Top-level, single entry point of the tree. |
| **Internal node** | Mid-level node used for routing/navigation only. |
| **Leaf node** | Bottom-level node with no children; where B+ tree data actually lives. |
| **Child pointer** | Pointer used to navigate from a node to a child during traversal (distinct from data pointer). |
| **Split** | When a full node breaks into two upon insertion, pushing a key up to the parent. |
| **Thrashing (index)** | Repeated, scattered random-access traversals instead of sequential reads — the B-tree range-query problem. |
| **Leaf linking** | B+ tree leaves connected in sorted order, enabling sequential range scans. |
| **LSM tree** | Log-Structured Merge tree — write-optimized alternative structure used by some engines (e.g., RocksDB, Cassandra) instead of B+ trees. |
| **Secondary index** | Any index not on the primary key (e.g., on `email`). |

---

## Quick Revision (skim before an interview)

1. Full table scan = read every page; cost is measured in pages, not rows.
2. B-tree node = one disk page. Fan-out (M) = how many elements fit per page, driven by key+value width.
3. High fan-out → shallow tree → few IOs (real example: ~500 fan-out → 100M rows in 3-4 levels).
4. B-tree limitations: (1) values waste space in nodes that don't need them, (2) range queries thrash due to random key placement, (3) hard to cache wide-keyed trees in memory.
5. B+ tree fix: values pushed to leaves only; internal/root nodes hold keys only (slim, high fan-out).
6. B+ tree keys are duplicated by design — routing copy (no value) above, real entry (with value) at leaf.
7. B+ tree search is always full-depth (no shortcuts) but that's a net win because the tree is shallower overall.
8. B+ tree leaves are (typically) linked in sorted order → range queries become sequential walks, not scattered traversals.
9. Nearly all relational DBs default to B+ trees, but call them "B-tree" in docs/commands.
10. Index depth/performance is per-index, not per-table — driven by key width and row count. Narrow fixed-width keys (BIGINT) beat wide/variable keys (UUID, VARCHAR, BLOB) for fan-out.

---

## Interview Questions & Answers

**Q1: What's the difference between a B-tree and a B+ tree?**
A: In a B-tree, keys and their values (data pointers) can exist at any level — root, internal, or leaf — each key stored exactly once. In a B+ tree, only leaf nodes store actual values; root and internal nodes store only routing keys (which are duplicates of leaf keys, minus the value). This makes B+ tree internal nodes much smaller, increasing fan-out and reducing tree depth, while also enabling fast range scans via linked leaves.

**Q2: Why do databases call their index type "B-tree" when they actually implement B+ trees?**
A: It's a long-standing naming convention/historical artifact — Postgres's index type is literally named `btree`, MySQL's is `BTREE`, but structurally what's implemented stores values only at leaves with linked leaf nodes, which is the B+ tree design. No major production database uses the pure, original 1970-paper B-tree.

**Q3: Why is a B+ tree's worst-case search cost (always full tree depth) actually not worse than a B-tree in practice?**
A: Because B+ tree internal nodes only store keys (no values), they're much smaller per element, which dramatically increases fan-out. Higher fan-out means the tree itself is shallower overall. So even though every B+ tree search must go all the way to a leaf (no "lucky" shortcut), that full traversal is shorter than the equivalent worst-case traversal in a B-tree holding the same data.

**Q4: Why are range queries slow on a plain B-tree but fast on a B+ tree?**
A: In a B-tree, even logically adjacent keys (e.g., 4–9) can be scattered across different nodes depending on insertion history, forcing separate, scattered traversals for each value ("thrashing"). In a B+ tree, all actual data lives at the leaf level in sorted order, and leaves are typically linked to each other — so once you find the starting key, you walk sideways along the linked leaves sequentially instead of re-traversing the tree for each value.

**Q5: How does an indexed column's data type affect index performance?**
A: Index depth and fan-out are driven by key width. A narrow, fixed-size key (e.g., 8-byte BIGINT) lets far more elements fit per page than a wide key (UUID = 16 bytes, VARCHAR = variable/large, BLOB = very large) — wider keys mean fewer elements per page, lower fan-out, more tree levels, and more IOs per search. This is also why composite indexes (multiple columns) have somewhat lower fan-out than single-column indexes.

**Q6: Why are random UUIDs considered bad for primary key indexes?**
A: Two compounding reasons. First, structurally: UUIDs are wider (16 bytes) than something like a BIGINT (8 bytes), reducing fan-out and increasing tree depth. Second, behaviorally: random UUIDs insert in random order rather than sequential order, which scatters writes across pages instead of filling them in order — this triggers far more page splits over time than sequential/auto-increment keys would, increasing write amplification and fragmentation.

**Q7: What is the difference between how Postgres and MySQL/InnoDB secondary indexes point to row data?**
A: In Postgres, a secondary index's data pointer points directly to the tuple (the row's physical storage location). In MySQL/InnoDB (and Oracle), a secondary index's data pointer points to the primary key value instead — meaning a lookup via a secondary index requires an extra step (look up primary key → then look up the row via the primary key's clustered index). This difference was a factor in Uber's well-known migration from Postgres to MySQL.

**Q8: Does every table have a B+ tree?**
A: No — every *index* has its own separate B+ tree, not every table. A table with no indexes defined is just an unordered heap of pages, which can only be searched via a full table scan. A table can also have multiple indexes (primary key + several secondary indexes), each maintaining its own independent B+ tree structure.

**Q9: Are all database indexes B+ trees?**
A: No. B+ tree is the default and most common index type in relational databases (e.g., Postgres's `btree`), but databases often support other index types for specific use cases — hash indexes, GIN/GiST for full-text search and JSON, BRIN for large naturally-ordered tables. Additionally, some storage engines optimized for write-heavy workloads (e.g., RocksDB-based engines like MyRocks, or Cassandra) use LSM trees instead of B+ trees entirely.

**Q10: In a real production B+ tree, roughly how many IOs does it take to find a row in a table with 100 million rows, and why?**
A: Typically just 3-4 IOs. Because each node corresponds to one disk page (e.g., 8 KB in Postgres) and can hold hundreds of keys (often 400-500+ for a narrow key type), the fan-out per level is roughly 500x. This means even 100+ million rows can be represented in a tree only 3-4 levels deep, and tree depth directly equals worst-case IOs since each level traversed costs one page read.

**Q11: Why does moving values out of internal nodes (B-tree → B+ tree) improve performance, mechanically?**
A: Internal nodes are read purely for navigation/routing — their values are never actually used during traversal, only the keys are compared. By removing the value/data-pointer payload from internal nodes, each element becomes much smaller, so far more keys fit in a single page. This raises the fan-out, which reduces the number of levels needed to represent the same dataset, which reduces the number of IOs required for any search.

**Q12: What's the tie-breaker rule when a search key exactly matches a routing key in a B+ tree?**
A: Go right. The traversal rule is: less-than goes left, between goes to the middle/relevant child, and greater-than-or-equal-to goes right.
