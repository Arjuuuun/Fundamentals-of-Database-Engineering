# Section 8: Replication

## What is Replication?

Replication is the process of copying data from one database server (the **master/primary**) to one or more other servers (**standbys/replicas**). The goal is horizontal scalability, high availability, and geographic distribution of reads.

---

## 8.1 Master/Standby Replication (Primary/Replica)

### Architecture

One **master node** accepts all writes — both DML and DDL:

- **DML (Data Manipulation Language)** — `INSERT`, `UPDATE`, `DELETE`. Row-level changes.
- **DDL (Data Definition Language)** — `CREATE TABLE`, `ALTER TABLE`, `CREATE DATABASE`. Schema-level changes.

Multiple **standby nodes** receive writes from master via a persistent TCP connection. Clients never write directly to standbys.

```
Client → WRITE → [Master 🔴]
                      ↓ WAL replication
              [Standby 🟢] [Standby 🟢] [Standby 🟢]
Client → READ → any of the above
```

### Why it's simple to implement

No write conflicts. Only one node ever accepts writes, so you never have two nodes disagreeing about the state of a row. This is what makes master/standby the default production choice.

### Reads

- **Master** → always fresh, always consistent
- **Standby** → might be slightly behind (replica lag)
- Standbys are **read-only** — any write attempt is rejected at the database level

### Horizontal Read Scaling

Reads can be distributed across as many replicas as you need. Classic pattern: replicas in different geographic regions serving local read traffic.

**Key advice:** keep your application and its database in the same region. Database protocols (including Postgres) are extremely chatty over TCP — acknowledgements, packet splits, round trips all compound over long distances.

### Eventual Consistency

When you write to master, the change takes time to propagate to replicas. During that window, a client reading from a replica sees **stale data**. Eventually all replicas converge — this is **eventual consistency**.

> ⚠️ This is not just a NoSQL concept. A fully relational, ACID-compliant Postgres setup with async replication still has eventual consistency on standby reads. ACID applies *within a single node's transactions*, not across nodes by default.

---

## 8.2 Multi-Master Replication

### Architecture

Multiple master nodes, each accepting writes simultaneously. Each master replicates to its own standbys.

```
Client A → WRITE → [Master 1 🔴] → [Standby] [Standby]
Client B → WRITE → [Master 2 🔴] → [Standby] [Standby]
```

### Why you'd want it

Master/standby hits a ceiling on write throughput — one master can only handle so many writes/sec. Multi-master distributes write load.

### The core problem: Write Conflicts

When two nodes independently accept writes to the same row, they can contradict each other. Someone has to resolve this — **conflict resolution** — and there's no clean universal answer:

- Last-write-wins → data loss
- Application-level merge → complex, domain-specific
- Reject one write → bad UX

### Industry reality

Most production systems avoid multi-master. Hussein's personal stance: optimize single-master writes as far as possible, scale reads with replicas, only consider multi-master as a genuine last resort. The conflict resolution complexity is rarely worth it.

---

## 8.3 Synchronous vs Asynchronous Replication

### The core question

After a client writes to master — when exactly does the master unblock the client and say "done"?

---

### Synchronous Replication

Master waits for confirmation from standby replica(s) **before** unblocking the client.

```
Client → WRITE → [Master]
                    ↓ (waits for replica ACK...)
              [Standby confirms]
                    ↓
         ← ACK back to client
```

**Guarantee:** once you get the ACK, the data exists on master AND at least one (or more) replica. **Strong consistency** — no eventual consistency.

**Cost:** higher write latency. Client blocks waiting for network round trips to replicas.

#### The "which replicas?" problem

If you have 7 standbys, waiting for all 7 is overkill. Postgres lets you tune this:

```sql
-- Wait for at least 1 of these standbys
synchronous_standby_names = 'FIRST 1 (standby1, standby2, standby3)'

-- Wait for any 2
synchronous_standby_names = 'FIRST 2 (standby1, standby2, standby3)'
```

- Standbys listed first have higher priority
- This is the same concept as **quorum** in Cassandra

**Cluster** — the term for the whole group of nodes (master + standbys) treated as one logical unit. Fully synchronous across all nodes ≈ single logical cluster.

---

### Asynchronous Replication

Master writes, WAL is flushed, client is unblocked immediately. Replication to standbys happens in the background.

```
Client → WRITE → [Master]
         ← ACK immediately
                    ↓ (background process, later...)
              [Standby gets WAL update]
```

**Gain:** faster write throughput, client isn't waiting on network hops.

**Cost:** replica lag is real. Background replication processes consume CPU. If master crashes before replication completes — **potential data loss** on standbys.

This is the **default** in most databases.

---

### Sync vs Async Comparison

| | Sync | Async |
|---|---|---|
| Consistency | Strong | Eventual |
| Write latency | Higher | Lower |
| Data loss risk | None (once ACKed) | Yes (replica lag window) |
| Write throughput | Lower | Higher |
| Complexity | Tunable (quorum) | Simple |

---

## 8.4 Replication Pros & Cons

### Pros

**Horizontal read scaling**
Distribute reads across as many standbys as needed. This is the primary reason most teams add replicas.

**Region-based queries**
Place replicas close to users geographically. Reads served locally = low latency. DB protocols are TCP-chatty — cross-region round trips compound fast.

**Write scaling (multi-master)**
Technically possible. Practically painful. See 8.2.

---

### Cons

**Eventual consistency (async)**
Standbys lag behind master. Not always a problem — Instagram likes being off by 50 for a second is fine. Financial transactions are not. Know your domain.

**Slower writes (sync)**
More replicas you wait on = higher write latency. This is a config dial, not a fixed cost.

**Multi-master complexity**
Conflict resolution has no clean universal answer. Avoid unless you've exhausted everything else.

---

### Hussein's optimization ladder (write scaling)

> Optimize single master → scale reads with replicas → only consider multi-master when you've genuinely exhausted everything else.

Mirrors the sharding ladder from Section 6 — the theme across the course is: don't reach for complexity before you need it.

---

## 8.5 Demo — Postgres Replication Setup (Key Concepts)

### What was set up

Two Postgres 13 instances via Docker — master (port 5432), standby (port 5433), each with its own mounted data volume.

### Three things required for replication to work

**1. Master's `pg_hba.conf`** — allow replication connections from a specific user:
```
host replication postgres all md5
```

**2. Standby's `postgresql.conf`** — tell standby where to find master:
```
primary_conninfo = 'application_name=standby1 host=host.docker.internal port=5432 user=postgres password=postgres'
```
`application_name` = unique identifier for this standby node. Master references it by this name.

**3. `standby.signal` file** — create this empty file in standby's data directory. Its mere existence tells Postgres: "this instance is a read-only standby."

### Initial data copy requirement

Standby must start with an **identical copy** of master's data directory. In production use `pg_basebackup`. In the demo: stop both instances, `cp -r master_data standby_data`.

If they start from different states → replication errors immediately.

### How WAL replication actually works

The standby doesn't receive SQL statements. It receives **WAL (Write-Ahead Log) segments** from master and replays them. The base copy + WAL replay = eventual identical state.

Confirmation in logs:
```
# Master log:
standby1 is now a synchronous standby with priority 1

# Standby log:
database system is ready, streaming WAL from primary
```

### Health check query

Run on master to verify replication is working:
```sql
SELECT * FROM pg_stat_replication;
-- Shows: connected standbys, application_name, replication mode (streaming/synchronous), lag
-- If empty → no active replication
```

### What the demo proved

| Action on master | Effect on standby |
|---|---|
| `CREATE TABLE` | Immediately appears |
| `INSERT` / `TRUNCATE` | Immediately replicated |
| `ALTER TABLE ADD COLUMN` | Immediately appears |
| Write attempt on standby | Error: `cannot execute on a read-only transaction` |

Read-only enforcement is at the Postgres level — not application level.

---

## Glossary

| Term | Definition |
|---|---|
| **Master / Primary** | The single node that accepts all writes in a replication setup |
| **Standby / Replica** | Read-only node that receives and replays changes from master |
| **DML** | Data Manipulation Language — `INSERT`, `UPDATE`, `DELETE` |
| **DDL** | Data Definition Language — `CREATE TABLE`, `ALTER TABLE`, `CREATE DATABASE` |
| **WAL (Write-Ahead Log)** | Log of all changes; what Postgres actually streams to replicas |
| **Synchronous replication** | Client waits for replica ACK before commit completes |
| **Asynchronous replication** | Client ACKed immediately; replication happens in background |
| **Replica lag** | Time gap between master write and standby receiving it |
| **Eventual consistency** | All nodes converge to same state eventually, but not instantly |
| **Strong consistency** | All reads reflect the most recent write immediately |
| **Quorum** | Minimum number of nodes that must confirm a write for it to succeed |
| **Cluster** | The full group of master + standby nodes treated as one logical unit |
| **Multi-master** | Multiple nodes accepting writes simultaneously; requires conflict resolution |
| **Conflict resolution** | Logic to decide the "true" value when two nodes have diverging writes |
| **pg_hba.conf** | Postgres host-based authentication config — controls who can connect and how |
| **primary_conninfo** | Standby config string telling it where and how to connect to master |
| **standby.signal** | Empty file whose presence marks a Postgres instance as a read-only standby |
| **pg_basebackup** | Production tool for creating a base backup of master for standby initialization |
| **pg_stat_replication** | System view on master showing all connected standbys and their replication status |

---

## Quick Revision

- **Master/standby** = one writer, many readers, no conflicts, simple
- **Multi-master** = multiple writers, conflict resolution required, avoid
- **Sync replication** = strong consistency, higher latency, tunable via quorum
- **Async replication** = eventual consistency, higher throughput, default
- **WAL streaming** = what actually moves between master and standby (not SQL)
- **standby.signal** = the file that makes a Postgres instance read-only
- **pg_stat_replication** = how you verify replication is alive on master
- Eventual consistency exists in relational DBs too — it's not just a NoSQL thing

---

## Interview Q&A

**Q: What is the difference between master/standby and multi-master replication?**
A: Master/standby has one node accepting writes, replicated to read-only standbys — simple, no conflicts. Multi-master allows multiple nodes to accept writes simultaneously but requires conflict resolution logic, making it significantly more complex and error-prone.

**Q: Is Postgres eventually consistent?**
A: It depends on configuration. A single node or synchronous replication setup gives strong consistency. With async replication, standbys may lag behind master — that's eventual consistency. ACID guarantees apply within a single node's transactions, not across nodes by default.

**Q: What's the difference between sync and async replication?**
A: Sync waits for replica ACK before unblocking the client — strong consistency but higher write latency. Async unblocks immediately and replicates in background — higher throughput but replica lag means eventual consistency and potential data loss if master crashes.

**Q: How does Postgres replication actually work under the hood?**
A: The standby connects to master and streams WAL (Write-Ahead Log) segments. It replays those WAL entries on its own copy of the data. This is why the standby must start from an identical base copy of master's data directory.

**Q: How would you configure which standbys to wait for in synchronous replication?**
A: Via `synchronous_standby_names` in master's `postgresql.conf`. You can specify `FIRST N (standby1, standby2, ...)` to wait for N standbys, with priority given to standbys listed earlier.

**Q: When would you use multi-master replication?**
A: Only when write throughput has genuinely exhausted all other options — better indexes, partitioning, caching, connection pooling. The conflict resolution complexity makes it a last resort.

**Q: What is replica lag and why does it matter?**
A: The time gap between a write being committed on master and appearing on a standby. During this window, reads from standbys return stale data. For low-stakes data (social likes, view counts) this is fine. For financial or inventory data, you'd either read from master or use synchronous replication.
