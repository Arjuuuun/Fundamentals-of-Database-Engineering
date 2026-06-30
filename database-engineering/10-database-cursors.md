# 10 — Database Cursors
> Section 10 | Fundamentals of Database Engineering | Hussein Nasser

---

## Why Cursors Exist — The Problem First

Imagine you have a table with 1 million students and you want all students who scored between 90 and 100. Let's say that's still 500,000 students.

If you run a normal query:

```sql
SELECT * FROM grades WHERE grade BETWEEN 90 AND 100;
```

Here is what happens step by step:

```
1. Postgres builds a query plan (which index to use)
2. Postgres scans the table and finds all 500,000 matching rows
3. Postgres compiles the full result
4. All 500,000 rows travel across the network to your Python app
5. Your Python app holds ALL 500,000 rows in RAM
6. Only NOW can your app do anything with the data
```

**The problems:**
- Your Python app needs enough RAM to hold 500,000 rows
- You waited for ALL rows before seeing even one
- If you only needed the first 50 rows, you wasted everything

**Cursors solve this** — they let you say: "Build the plan, but don't fetch anything yet. I'll ask for rows when I'm ready, in small chunks."

---

## What a Cursor Actually Is

A cursor is a **pointer** that sits on the database server, pointing at a result set. You ask it for rows one chunk at a time. The rows come to your app only when you ask.

Think of it like a book:
- **Normal query** = photocopying the entire book and handing it to you
- **Cursor** = giving you a bookmark in the library. You go read a page, come back, read another page.

---

## Two Types of Cursors

### Type 1: Client-Side Cursor (default)

When you create a cursor in Python without a name:

```python
cur = conn.cursor()  # no name = client-side
cur.execute("SELECT * FROM employees")
# ALL rows are now in Python's memory
rows = cur.fetchmany(50)  # just picking 50 from what's already in RAM
```

**What happens under the hood:**
```
Python says "execute this query"
→ Postgres runs the full query
→ ALL results travel across the network
→ ALL results sit in Python's RAM
→ fetchmany(50) just picks 50 from local memory (instant)
```

The word "cursor" here is misleading — it's really just Python's connection object. The data is already on the client side.

**Hussein's demo numbers:**
- Execute query: **845ms** (pulling 1M rows across network)
- Fetch 50 rows: **~0ms** (already in RAM, just picking 50)

---

### Type 2: Server-Side Cursor

When you create a cursor in Python **with a name**:

```python
cur = conn.cursor(name='my_cursor')  # name = server-side
cur.execute("SELECT * FROM employees")
# Nothing fetched yet — plan built on server, data stays there
rows = cur.fetchmany(50)  # NOW 50 rows travel across network
```

**What happens under the hood:**
```
Python says "execute this query"
→ Postgres builds the query plan only
→ Nothing travels across the network
→ Python's RAM stays empty

Python says "fetchmany(50)"
→ Postgres fetches 50 rows
→ 50 rows travel across network
→ Python holds only 50 rows in RAM
```

**Hussein's demo numbers:**
- Execute query: **3ms** (just building the plan, nothing fetched)
- Fetch 50 rows: **~2ms** (50 rows travel across network — round trip)

---

## Side by Side Comparison

```
CLIENT-SIDE                          SERVER-SIDE
-----------                          -----------
execute → 845ms                      execute → 3ms
  (1M rows cross network)              (just plan, nothing moves)

fetchmany(50) → ~0ms                 fetchmany(50) → ~2ms
  (already in RAM)                     (50 rows cross network)

Total for 50 rows: 845ms             Total for 50 rows: 5ms
```

**If you only need 50 rows from 1 million — server-side wins massively.**

---

## Server-Side Cursor: The SQL Behind It

In raw Postgres SQL, server-side cursors look like this:

```sql
BEGIN;  -- must be inside a transaction

DECLARE my_cursor CURSOR FOR
  SELECT * FROM grades WHERE grade BETWEEN 90 AND 100;
-- Query plan built. Nothing fetched yet.

FETCH 50 FROM my_cursor;   -- fetch first 50 rows
FETCH 50 FROM my_cursor;   -- fetch next 50 rows
FETCH 50 FROM my_cursor;   -- fetch next 50 rows

COMMIT;  -- closes the cursor
```

In Python (psycopg2), the `name='my_cursor'` parameter does all of this automatically behind the scenes.

---

## Pros and Cons

### Client-Side Cursor

**Pros:**
- Server is relieved after sending data — no ongoing memory cost on DB
- Works perfectly with stateless REST APIs
- Works with horizontal scaling (any server can handle any request)
- Fetching from local RAM is instant after the initial load

**Cons:**
- Entire result set travels across network — can saturate bandwidth
- Your app needs enough RAM to hold everything
- You wait for ALL rows before seeing the first one

---

### Server-Side Cursor

**Pros:**
- App memory stays flat — only holds what you asked for
- First rows available almost instantly (3ms vs 845ms)
- Can cancel mid-way — processed 100k rows and done? Just close the cursor
- Perfect for batch processing and ETL pipelines

**Cons:**
- **Stateful** — tied to one specific database connection on one specific server. Cannot share across servers.
- **Long-running transaction** — cursor must stay inside a transaction. Long transactions block other operations on the table.
- **Cursor leaks** — if your code crashes before closing the cursor, Postgres holds that memory until timeout. At scale, thousands of leaked cursors can crash your database.
- Cannot use in stateless REST APIs easily (request 1 opens cursor on Server A, request 2 hits Server B — cursor is gone)

---

## Cursor Leaks — The Most Dangerous Part 🔴 CORE

If your code crashes between opening and closing a server-side cursor, Postgres keeps holding memory for it. This is called a **cursor leak**.

```python
# DANGEROUS — if an error happens, cursor never closes
cur = conn.cursor(name='my_cursor')
cur.execute("SELECT * FROM employees")
rows = cur.fetchmany(100)
do_something_that_might_crash(rows)  # if this throws...
cur.close()   # ...this never runs. Leak!
conn.close()  # ...this never runs. Leak!

# SAFE — finally block always runs, even on exceptions
cur = conn.cursor(name='my_cursor')
try:
    cur.execute("SELECT * FROM employees")
    rows = cur.fetchmany(100)
    do_something_that_might_crash(rows)
finally:
    cur.close()   # always runs
    conn.close()  # always runs
```

This is the same pattern as connection pooling — `client.release()` in a `finally` block. Same discipline, same reason. Always clean up your resources.

---

## When to Use Which

| Situation | Use |
|---|---|
| REST API endpoint | Client-side + WHERE + LIMIT |
| Infinite scroll / pagination | Client-side + keyset pagination |
| Batch processing 1M rows | Server-side cursor |
| ETL pipeline | Server-side cursor |
| Streaming results to WebSocket | Server-side cursor |
| Stored procedures (PL/pgSQL) | Server-side cursor |

**Hussein's personal preference:** Client-side for web apps. Just be disciplined — always use a WHERE clause and LIMIT. Never do unbounded `SELECT *` queries.

---

## Cursors vs Keyset Pagination

These are not really competitors — they solve problems at different layers.

| | Server-Side Cursor | Keyset Pagination |
|---|---|---|
| DB cost per page | Low (query runs once) | Higher (fresh query each time) |
| Works with REST APIs | No (stateful) | Yes (stateless) |
| Works across multiple servers | No | Yes |
| Long transaction risk | Yes | No |
| Best for | Batch/ETL | Web APIs |

Keyset pagination re-executes a query on every page request. Cursors execute once and iterate. But cursors don't work in stateless REST APIs — so keyset pagination is the pragmatic choice for web.

---

## Glossary

| Term | Definition |
|---|---|
| Cursor | A pointer to a result set. Lets you fetch rows incrementally instead of all at once. |
| Client-side cursor | Default cursor in psycopg2. All rows pulled to Python memory on execute. |
| Server-side cursor | Named cursor in psycopg2. Data stays on Postgres server, Python fetches in chunks. |
| Stateful | Tied to a specific connection/server. Cannot be shared or moved. |
| Cursor leak | Forgetting to close a server-side cursor. Postgres holds memory until timeout. |
| Long-running transaction | A transaction that stays open for a long time. Blocks DDL, MVCC cleanup, certain writes. |
| ETL | Extract, Transform, Load. Moving data from one system to another, often in bulk. |
| Round trip | One request to the server and one response back. Each fetchmany() on a server-side cursor costs one round trip. |
| Unbounded query | A query with no LIMIT or WHERE clause. Returns everything. Always avoid. |

---

## Quick Revision

- Normal query = all rows travel to client at once = memory spike
- Client-side cursor = same thing, just Python's name for its connection object
- Server-side cursor = data stays on Postgres, Python fetches in chunks
- One word difference in psycopg2: `conn.cursor(name='anything')`
- Client-side: execute is slow (845ms), fetch is instant (data already in RAM)
- Server-side: execute is fast (3ms), fetch costs a round trip (~2ms per chunk)
- Server-side cursors must live inside a transaction → long transaction risk
- Always close cursors in a `finally` block → prevents leaks
- Cursors are stateful → don't use in REST APIs → use keyset pagination instead
- Hussein's rule: web apps → client-side + WHERE + LIMIT. Batch jobs → server-side cursor.

---

## Interview Q&A

**Q: What is a database cursor?**
A cursor is a pointer to a result set on the database server. Instead of pulling all rows at once, you declare a cursor and fetch rows incrementally in chunks. This saves client memory and lets you start processing data before the full result is ready.

**Q: What's the difference between client-side and server-side cursors?**
Client-side cursor: execute pulls all rows across the network into application memory immediately. fetchmany() just picks from local RAM. Server-side cursor: execute builds the query plan but fetches nothing. Data stays on Postgres. fetchmany() triggers a network round trip to fetch that chunk. Client-side is simpler and works with REST. Server-side saves memory and is better for batch processing.

**Q: Why can't you use server-side cursors in a REST API?**
Server-side cursors are stateful — tied to one specific database connection on one specific server. In a horizontally scaled REST system, request 1 might open a cursor on Server A, but request 2 for the next page might hit Server B which has no knowledge of that cursor. Use keyset pagination instead — it's stateless and works with any server.

**Q: What is a cursor leak and why is it dangerous?**
A cursor leak happens when a server-side cursor is opened but never closed — usually because the code crashed before the close() call. Postgres keeps holding memory for the cursor until it times out. At scale, thousands of leaked cursors can exhaust database memory and crash it. Always close cursors in a finally block.

**Q: What are the risks of long-running cursors?**
Server-side cursors must live inside a transaction. A slow iteration through millions of rows means a long-running transaction. This blocks DDL operations like ALTER TABLE, prevents Postgres MVCC from cleaning up dead row versions (table bloat), and can hold locks that block writes.

**Q: When would you use a server-side cursor over keyset pagination?**
Server-side cursor for batch processing, ETL pipelines, or any single-process job that needs to iterate through millions of rows efficiently without re-executing the query each time. Keyset pagination for web APIs and anything that needs to scale across multiple servers.
