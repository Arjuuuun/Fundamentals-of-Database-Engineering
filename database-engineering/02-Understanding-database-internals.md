# Storage Fundamentals — How Tables & Indexes Live on Disk

> Source: Hussein Nasser — Fundamentals of Database Engineering
> Section: Storage — Pages, Heap, Indexes, Row vs Column, Primary vs Secondary Keys

---

## 1. The Core Building Blocks

### Row ID — The Hidden Identity
Every row has an internal, system-generated identifier separate from your primary key.
- **Postgres**: called `ctid` (tuple ID) — system-maintained, all indexes point to this
- **MySQL**: the primary key *becomes* the row ID (no separate internal identifier)
- This is what indexes actually point to, not your business columns

### Page — The Real Unit of Storage
> A page is a fixed-size block of bytes — the smallest unit the database reads/writes from disk.

```
Postgres default: 8 KB per page
MySQL default:    16 KB per page
(both configurable)
```

A page holds **multiple rows** (however many fit given the row width). The database **never reads one row** — it reads a whole page and you get everything in it whether you wanted it or not.

### IO — The Currency of the Database
> An IO is a disk read/write operation. **1 IO = 1 or more whole pages. Never a single row. Never a single column.**

This is the most important mental model in this entire section:

```
You ask for: employee.name WHERE id = 40
Database does: IO → fetches WHOLE PAGE containing row 40
              → page also contains rows 41, 42
              → DB discards them in memory, gives you only row 40's name
              → you paid full IO cost for a full page regardless
```

- IO count is **the metric** of query performance — fewer IOs = faster query
- Postgres IOs often hit the **OS page cache** (memory), not physical disk — Postgres relies on this heavily
- `SELECT *` and `SELECT name` cost the same IO — both fetch the whole page. The difference is only in deserialization and network transfer.

### Heap — Where All Your Data Lives
> The heap is the full collection of pages containing your table's actual data — all rows, all columns, typically unordered.

- Searching the heap without guidance = full table scan (reading every page sequentially)
- For a 10,000-row table split across 333 pages, finding one row at the end = 333 IOs
- This is the thing indexes exist to avoid

### Index — The Shortcut Into the Heap
> A separate data structure (typically a B-tree) containing only indexed column(s) + a pointer (row ID + page number) to the actual row in the heap.

```
Index on employee_id:
  10    → page 0, row_id 1
  20    → page 0, row_id 2
  ...
  10000 → page 333, row_id 1000
```

**Standard two-hop lookup:**
```
Hop 1: IO on index → traverse B-tree → "employee 40 is at page 1, row_id 4"
Hop 2: IO on heap  → fetch page 1 → extract row 4 → discard rows 5, 6 (same page)
```

Two IOs vs potentially hundreds for a full scan.

**Indexes aren't free:**
- Stored as pages on disk — cost IO to read themselves
- Must fit in memory to be useful (large indexes that spill to disk are expensive to search)
- Every write to the table must also update every index on it — more indexes = slower writes

---

## 2. Row-Oriented vs Column-Oriented Storage

Same table, two completely different physical layouts. No indexes in this analysis — pure storage behavior.

```sql
-- Test queries:
Q1: SELECT first_name FROM employees WHERE ssn = 666;     -- selective filter, one column
Q2: SELECT * FROM employees WHERE id = 1;                  -- fetch whole row
Q3: SELECT SUM(salary) FROM employees;                     -- aggregate, one column
```

---

### Row-Oriented Storage (Row Store)

> Rows laid out contiguously on disk: `id, first_name, last_name, ssn, salary, dob...` then immediately the next row's full data.

```
Block 1: [1,John,Smith,111,90k,...] [2,Melissa,Park,222,95k,...]
Block 2: [3,Rick,...,333,...]       [4,Paul,...,444,...]
Block 3: [5,Hussein,...,555,...]    [6,...,666,...]
```

One IO = one block = multiple *complete* rows with ALL columns.

**Q1 (SELECT first_name WHERE ssn=666):**
```
Read Block 1 → ssn = 111, 222 → no match → discard
Read Block 2 → ssn = 333, 444 → no match → discard
Read Block 3 → ssn = 666 → MATCH → first_name already in memory (same block) → return free
```
3 IOs. Extra columns from matched row = zero cost (already loaded).

**Q2 (SELECT * WHERE id=1):**
```
Read Block 1 → id=1 found → ALL columns already in memory → return immediately
```
Cheap. `SELECT *` is the NATURAL shape of row storage.

**Q3 (SELECT SUM(salary)):**
```
Read every block → each block gives salary BUT ALSO all other columns
→ paying full IO cost for all columns just to use the salary column
→ massive wasted reads
```
Weak point of row stores — aggregating one column from a wide table.

---

### Column-Oriented Storage (Columnar / Column Store)

> Columns laid out contiguously on disk: all `id` values together, all `first_name` values together, etc. Each value tagged with its row ID.

```
ID col:        [1001:1][1002:2][1003:3][1004:4][1005:5][1006:6]
FirstName col: [1001:John][1002:Melissa][1003:Rick]...
SSN col:       [1001:111][1002:222]...[1006:666]
Salary col:    [1001:90k][1002:95k]...
```

Row ID is **duplicated in every column** — the glue for reconstructing full rows.

**Q1 (SELECT first_name WHERE ssn=666):**
```
Read ONLY SSN column blocks → scan → find 666 → row_id = 1006
Jump to FirstName column → read block for row_id 1006 → return
(Never touched last_name, salary, DOB, title, join_date at all)
```
~3 IOs. But critically: only 2 out of 8 columns touched.

**Q2 (SELECT * WHERE id=1):**
```
Read ID col     → find row_id 1001 → 1 IO
Read FirstName col → 1 IO
Read LastName col  → 1 IO
Read SSN col       → 1 IO
Read Salary col    → 1 IO
Read DOB col       → 1 IO
Read Title col     → 1 IO
Read JoinDate col  → 1 IO
```
8 IOs for ONE ROW. **`SELECT *` is catastrophic on column stores.**

**Q3 (SELECT SUM(salary)):**
```
Read ONLY Salary column → 1-2 blocks for entire table → done
```
Might be 1-2 IOs for the ENTIRE TABLE. This is why columnar databases exist.

**Compression bonus:**
Similar values stored adjacently compress extremely well. Three people earning 100k → stored once as `100k:[1003,1004,1005]`. Row stores can't do this — each row is a unique mix of different types.

---

### Pros & Cons Summary

| | Row-Oriented | Column-Oriented |
|---|---|---|
| **Designed for** | OLTP — frequent reads/writes of individual records | OLAP — aggregations across many rows, few columns |
| **Writes** | Fast — single contiguous write per row | Slow — every column structure must be updated |
| **`SELECT *`** | Natural / cheap | Catastrophic — N separate column reads |
| **Aggregates (SUM/AVG/COUNT)** | Expensive — full scan pulls unused columns | Extremely efficient — touch only relevant columns |
| **Multi-column queries** | Efficient — all columns loaded together | Inefficient if touching many columns |
| **Compression** | Poor — rows are heterogeneous | Excellent — columns are homogeneous/repetitive |
| **Examples** | Postgres, MySQL, SQL Server, Oracle | ClickHouse, BigQuery, Redshift, Cassandra (partial) |

### Real-world pattern
**OLTP database (Postgres/MySQL) + OLAP data warehouse (Redshift/BigQuery/ClickHouse)** is a standard production architecture — transactional writes go to the row store, analytics queries run on the column store fed by ETL/CDC pipelines.

### Per-table storage engine (advanced)
Some databases let you choose storage engine per table — high-write transactional table → row store; read-only analytics table → column store. You generally **cannot JOIN across row and column store tables** efficiently.

---

## 3. Primary Key vs Secondary Key — Heap-Organized vs Index-Organized Tables

### Default — Heap-Organized Table (HOT)
Without a clustering primary key, a table has **no physical order**:
```
Insert 7 → goes to end
Insert 1 → goes to end  (NOT sorted)
Insert 2 → goes to end  (NOT inserted between 1 and 7)

Physical layout: [7, 1, 2, ...]  ← pure insertion order
```

### Primary Key = Clustering = Index-Organized Table (IOT)
When a database clusters on the primary key, the heap **physically reorders itself** around that key:
```
Insert 1 → physically first
Insert 8 → after 1
Insert 2 → must go BETWEEN 1 and 8 → page may need to make space (page split)
```
The database reserves gaps in pages for anticipated future values and uses page splits when needed — overhead on writes, but pays off in reads.

**Terminology across databases:**
| Database | Term | Behavior |
|---|---|---|
| Oracle | Index-Organized Table (IOT) | Optional — you choose |
| MySQL (InnoDB) | Clustered index | **Mandatory** — every table must have one (auto-generated if missing) |
| SQL Server | Clustered index | Optional (often defaults to PK) |
| **Postgres** | — | **No IOT** — all indexes are secondary; PK is just a secondary index |

**Range query payoff (the reason clustering exists):**
```sql
SELECT * FROM table WHERE id BETWEEN 1 AND 9;
```
With a clustered PK, rows 1-9 are **physically adjacent on disk** → possibly 1 IO for the whole range. Without clustering → rows scattered across heap → potentially N IOs.

### The UUID Primary Key Problem (revisited and explained)
```
MySQL (clustered PK) + UUID primary key:
  UUID 'a3f9...' → must be inserted at specific physical location (sorted order)
  UUID '02b1...' → completely different physical location
  UUID 'f821...' → yet another location

Every insert = potential page split, no cache locality, scattered writes
→ catastrophic write performance at scale
```
Fix: use **auto-increment integers** or **sequential/sortable IDs** (ULID, UUIDv7) for MySQL PKs. In Postgres this matters less — PK is secondary anyway, no physical reordering happens.

### Secondary Index
> The heap stays unordered. A **separate B-tree structure** maintains ordered lookup — the two live independently.

```
Heap (unordered):       [row7, row1, row300, row8, row2, ...]

Secondary index (B-tree, separate):
  1   → row_id X  (points into heap)
  2   → row_id Y
  7   → row_id Z
  8   → row_id W
  300 → row_id V
```

**The two-hop cost:**
```
Step 1: Search secondary index → get row_id
Step 2: Jump to heap using row_id → fetch actual row data
```

### 🔑 Postgres — Everything is a Secondary Index
In Postgres:
- **All indexes** (including PK) are secondary — they point to the heap's `ctid` (row ID)
- The heap is always heap-organized (no clustering by default)
- `CLUSTER` command exists to manually reorder the heap once, but it's not automatically maintained
- MVCC creates new row versions on every UPDATE → **every index** must be updated on every write (because row IDs change with new versions)

This is why write amplification in Postgres scales with the number of indexes — each index maintains its own pointer structure to the (potentially moved) row.

### HOT vs IOT — Mental Model
| Term | Meaning |
|---|---|
| **Heap-Organized Table (HOT)** | Heap is unordered — all indexes secondary — Postgres default |
| **Index-Organized Table (IOT)** | Heap is physically ordered around one key — MySQL InnoDB default, Oracle/SQL Server optional |

Oracle's naming wins here — "index-organized" is self-descriptive in a way "clustered index" isn't.

---

## Quick Revision Cheat Sheet ⚡

**IO fundamentals:**
- 1 IO = 1+ whole pages (never a single row/column)
- IO count = the metric of query performance
- Fewer IOs = faster query — everything else is an optimization in service of this

**Row vs Column — one-line decision:**
- Writing a lot / need individual records → **Row store**
- Reading a lot / aggregating few columns → **Column store**
- `SELECT *` → great for row, terrible for column

**Primary vs Secondary — one-line decision:**
- Clustered PK (MySQL/Oracle/SQL Server) → heap is ordered → great range queries, careful with random PKs
- Secondary (Postgres, all non-clustered indexes) → separate B-tree + heap lookup → two hops always

---

## Interview Pointers

**"Why is `SELECT *` bad?"**
Two reasons: (1) In row stores — unnecessary deserialization of all columns after the page fetch. (2) In column stores — each column is a separate IO; `SELECT *` multiplies IOs by number of columns. Always select only what you need.

**"What is a full table scan and when does it happen?"**
When the DB has no index to guide it, it reads every page in the heap sequentially to find matching rows. IO count = (table size) / (page size). Happens on unindexed columns, OR when the optimizer decides an index scan is actually more expensive (e.g., fetching 80% of a table is cheaper as a full scan).

**"What's the difference between OLTP and OLAP? Which storage model fits each?"**
OLTP (transactional, frequent reads/writes of individual rows) → row store. OLAP (analytical, aggregations over millions of rows, few columns) → column store. Real architectures often have both.

**"Why is UUID a bad primary key in MySQL?"**
MySQL InnoDB clusters the heap on the PK. Random UUIDs scatter inserts across the physical heap → constant page splits, no cache locality, severe write amplification. Use auto-increment or sequential IDs (ULID, UUIDv7) instead. Less critical in Postgres since PK is secondary (heap isn't reordered).

**"Why does Postgres update all indexes on every row change?"**
MVCC: updates create new row versions with new row IDs (ctid). All indexes point to row IDs, so every index must be updated to point to the new version. This is write amplification — minimize index count on write-heavy tables.

**"What is a clustered index?"**
An index where the heap itself is physically ordered according to that key — the table IS the index. MySQL PKs are always clustered. Postgres has no native clustering (manually via `CLUSTER` command only). Oracle calls this an Index-Organized Table (IOT).

**"Two queries run the same IO count — but one uses an index. Why might the indexed query be faster?"**
The index IOs are smaller and targeted (B-tree traversal = logarithmic page reads). The non-indexed query reads *every* page in the heap sequentially. Even if the IO count were similar by coincidence, the index avoids reading and deserializing irrelevant data.

---

*End of Storage Fundamentals section. Next section: [to be added]*
