---
layout: post
title:  "About Data Persistence"
date:   2026-04-01 13:09:00 +0000
categories: distributed-systems
---
## System Design from First Principles — Lecture 5 Persistence

### 1. Fundamental challenge
**Persistence is important** — you cannot afford to lose data. But disk is *slow*.
The goal: data that survives power outages *and* systems that feel fast.

#### Latency in Human Scale

To make latency intuitive, imagine scaling nanoseconds to human time:

| Storage   | Scaled Latency |
|-----------|---------------|
| L1 Cache  | 0.5 seconds   |
| RAM       | 2 minutes     |
| SSD       | 2 days        |
| HDD       | 5 months      |

> **Key insight:** We want persistence *and* speed. These two goals conflict — and the rest of this lecture is about how to reconcile them.

---

### 2. Databases & The OS Lie

#### Buffered I/O

When a process calls `write()`, the OS does **not** immediately write to disk. Data goes to the **page cache** (in RAM), and `write()` returns immediately. This is the "OS lie" — your data isn't on disk yet.

```
Process          Page Cache (RAM)         Disk
   │                    │                   │
   │──── write() ──────►│                   │
   │◄─── returns ───────│                   │
   │                    │── (eventually) ──►│
```

#### fsync() — Safe but Slow

`fsync()` blocks until writes actually reach disk and are confirmed. It is the antidote to the OS lie, but expensive — the process cannot proceed until the disk acknowledges.

> **Trade-off:** Buffered I/O (fast, unsafe) vs. `fsync()` (safe, slow). The choice depends on how much data loss your use-case can tolerate.

---

### 3. Write-Ahead Log (WAL)

Instead of writing directly to tables (random I/O), databases **append every write to the end of a log file** first — the Write-Ahead Log.

```
DB Write ──► Append to WAL ──► ACK returned to client
                  │
                  │ (later, when idle)
                  ▼
            Update index/tables
```

WAL converts random writes into **sequential writes**, which are dramatically faster on both HDDs and SSDs.

> **Sequential writes > random writes.**
> On HDDs, the read/write head needs to physically move — appending keeps the head still. SSDs benefit from sequential patterns too.

---

### 4. SSD Internals — Write Amplification

A common misconception: **SSD is not fast RAM**. You cannot overwrite or delete a single byte on an SSD. You can only erase in large chunks (~2 MB blocks).

#### The Read-Modify-Erase-Write Dance

To change **1 byte** on an SSD:

```
1. Read 2 MB chunk → RAM
2. Modify 1 byte in RAM
3. Erase the entire 2 MB block on SSD
4. Write the full 2 MB back to SSD
```

Goal: update 1 byte → actually moved 2 MB. This is **write amplification**.

#### Flash Translation Layer (FTL)

The **FTL** is a small orchestrator embedded in every SSD. It manages physical erase blocks, tracks logical-to-physical block mappings, and handles wear leveling — making the drive appear as a simple byte-addressable device and hiding all the complexity above.

---

### 5. B-Tree — Shallow and Fat

Without an index, every query is a full table scan. The **B-Tree** solves this: optimized for reads, keeping itself shallow by making each node very wide (many children).

#### Exponential Growth (fanout = 500)

```
Layer 1 (Root):  1 node
Layer 2:         500 nodes
Layer 3:         250,000 nodes   → 125 million pages
Layer 4:         125 million nodes → 62.5 billion items
```

**62.5 billion rows indexed with only 4 disk reads (~30–40 ms per lookup).**

---

### 6. LSM Tree — Log-Structured Merge Tree

The B-Tree is optimized for reads. The **LSM Tree** makes the opposite bet: optimize for writes. Used by **Cassandra**, **RocksDB**, and other NoSQL engines.

The key rule: *only ever do sequential writes. Never do random writes on disk.*

#### Write Path

```
Write arrives
     │
     ▼
MemTable (sorted list in RAM)  ──also writes──►  WAL on disk (durable)
     │
     │ (when MemTable is full → flush)
     ▼
SSTable on disk (immutable, sorted, sequential write)
```

#### Bloom Filter

Each SSTable has an associated **Bloom filter** — a probabilistic data structure that answers:
*"Is this key in this SSTable?"*

- **"Definitely not"** → skip the SSTable entirely (no disk read needed)
- **"Maybe yes"** → go check

If the Bloom filter says no, the SSTable is not even touched. This dramatically reduces unnecessary disk reads.

---

### 7. The RUM Conjecture

A fundamental trade-off in data structure design — you can optimise for any **two** of the three, but never all three simultaneously:

```
              R (Read)
             /        \
            /          \
  B-Tree ──/            \── B-Tree
  (read+mem)            (read+mem)
          /              \
         /    Pick Two    \
        /      Only        \
U (Update) ────────────── M (Memory)
    LSM Tree (update+read)
```

| Optimise for | Use          | Example workload |
|--------------|--------------|-----------------|
| R + M        | **B-Tree**   | Banking, relational DBs (read-heavy) |
| U + R        | **LSM Tree** | Logs, event streams, Cassandra (write-heavy) |

> **Rule of thumb:** If your application is read-heavy, use a B-Tree. If it will be write-heavy, use an LSM Tree.

---

### 8. The Invisible Enemy — Bit Rot

Even at rest, data can silently corrupt. Cosmic rays, voltage fluctuations, and magnetic interference can flip bits without the OS noticing. **Do not trust hardware.**

**Solution:** Checksums (e.g. SHA-256). Compute a hash on write; recompute and verify on read. ZFS does this automatically for every block.

#### Disk Failure Rates

Out of 10,000 disks, approximately **1 fails per month**. At scale, disk failures are expected daily events — not exceptional ones. Distributed systems must treat failure as the norm.

#### 2003 — Google File System (GFS)

GFS splits data into chunks, storing **3 copies across 3 machines on 3 different racks**. If one rack goes down, data survives on the other two.

But this introduces a **consistency problem**: all three replicas might have slightly different data at any moment. Consensus protocols like **Raft** solve this.

---

### 9. The Big Trade-Off Spectrum

Every persistence decision sits on a spectrum between maximum speed and maximum durability:

```
High risk / High speed ◄─────────────────────────────────► Low risk / Low speed

  RAM /           fsync() +       Distributed        S3 /
Buffered I/O        SSD           replication      Multi-region
```

#### Practical Decision Framework

**Is it a like on a post?**
→ It's okay if people see the count update with a 2–3 second delay.
→ Use an **LSM tree** with buffered I/O. Optimise for throughput.

**Is it a bank transfer?**
→ Data loss is unacceptable.
→ Use **`fsync()`** and wait for *all* replicas to reply before returning success. Optimise for durability.

---
