---
layout: post
title:  "Paper: Google File System — 2003"
date:   2026-04-17 00:00:00 +0000
categories: distributed-systems
---

## Abbreviations

| Symbol | Meaning |
|---|---|
| CS | Chunk Server |
| C | Chunk |
| NV | Non-volatile (survives master crash) |
| V | Volatile (lost on master crash, re-fetched on recovery) |
| P | Primary Chunk Server |
| S | Secondary Chunk Server |
| M | Master |


## Goals

- Distributed file system that stores **huge** data, files in Gigabytes
- Anybody within Google can read it, data guaranteed not to be interleaved with **concurrent atomic appends**
- Data is so large that **sharding is needed**
- Returns offset where written data ends
- **Automatic recovery** from failures

## Non-Goals

- Not distributed across the whole world — GFS is for a **single data center**
- **Internal use** only (not a general-purpose public FS)
- Optimised for **sequential access**, not low latency — **throughput is the focus**

> GFS does **not** guarantee strong consistency. It is **inconsistent** by design.
> The use case (internal web data pipelines) allows weak consistency.

---

## Architecture

Since file is huge, its not going to be stored as a single contigous object, we divide it in chunks. Reads and writes can be made parallel. 64MB chunks.  
Why 64MB size? Its a big chunk, so with such big chunks we will have less metadata for these chunks. If files are small, we will have fragmantation, downside! `:(`. Big chunk is suitable for sequential reads, we can keep open TCP connection.

Can a single master become the bottleneck? Probably not, because actual data is queried from CS, client caches response from master, chunk size is big so master won't be queried very much if files are big.

```
                    ┌──────────────────────────┐
                    │          Master          │
                    │  filename → CS location  │
                    └──────────────────────────┘
                       /          |          \ Heartbeat
                      /           |           \
                 ┌────┐        ┌────┐        ┌────┐
                 │ CS │        │ CS │        │ CS │
                 └────┘        └────┘        └────┘
CS = stores all actual data
Master = keeps track of mapping all **filenames to all chunk identifiers**, and all chunk identifiers and **chunk servers**. This is small data less than 64KB per mapping. Millions of mappings can be stored in memory.
Master does prefix compression on file names to conserve memory.
Heartbeat tells what chunks a CS stores.
```

**Read flow:** Client → Master (returns CS locations) → Client reads chunks directly from CS.

---

## Master Data — Two Tables

The master holds two tables in **memory**:

### Table 1: filename → chunk handles / chunk IDs
- Stored **non-volatile (NV)** — persisted to disk via operation log

### Table 2: chunk handle → (stored per chunk)
| Field | Volatile? |
|---|---|
| List of Chunk Servers holding this chunk | **Volatile** — re-fetched from CSes on recovery |
| Version number of each chunk | **Non-volatile** |
| Which CS is Primary | **Volatile** |
| Lease expiry of Primary | **Volatile** |

> **Why volatile fields?** The master can ask each CS on startup what chunks they hold.
> It would be wasteful to persist this — the ground truth lives on the CSes themselves.

## Master Durability — Operation Log + Checkpoint

- Master **appends a log** of every metadata update to disk → non-volatile
- Operation log must be made fault tolerant, so synchronously replicated to other nodes. (Could we use consensus?)
- Periodically writes a **checkpoint** (snapshot of full state)
- Log can be appended efficiently; disk limits total operations between checkpoints
- If master crashes → replay log from most recent checkpoint to restore state, checkpoint is implemented by making a file immutable after some operations and replicating that file.  

#### What If Master restarts
Master replays operation log to restore its state, state = filename-to-chunkIDs, version# of chunks
Chunk server locations are not stored on disk, so master will have to query all CS to get those back.  
Problem is repeating the operation log will take time, and querying CS will also take time. Checkpoint will help here, we will only have to replay after the last checkpoint, instead from the start of server's life.

---

## Read Operation

```
1. Client sends (filename, byte offset) → Master
2. Master returns chunk handle(e.g chunk 1) + list of CS locations (CS is A, C, D)
   → Client caches this result (avoids hitting master on every read)
3. Client contacts nearest CS with chunk handle + offset
   CS stores each chunk as a separate file on its local Linux FS
   Chunk files are named by their handle
4. CS returns the data, in case the CS gets outdated then client may receive stale data
```

---

## Write Operation (Record Append)

Goals are: How to write as fast as possible, how do we determine the ordering of cocurrent writes?
Answer is: Separating the ordering of writes from data transfer.

Data Transfer - master tells which CS to write to (say A, B and D), C writes to closest replica (say A) then A replica will send to other replicas (B and D). This data will be stored in memory for now.

Concurrent data - Problem arises when two clients write different data for same chunks. The only way to avoid this is to write to all replicas in the same order. C1 writing to A->B->C and C2 writing to C->B->A can mess up what is written. This is NOT allowed. Primary replica is needed which will **instruct the order**.

GFS is **append-only**. Clients append to named files; random overwrites are not the primary model.

### Step 1 — Find or Elect a Primary

Client asks Master: *"I want to append to this file — where should I go?"*
Master must return a **Primary CS**. 

**If no Primary exists:**
- Master finds the CS that holds the **most up-to-date replica**
- "Most up-to-date" = the CS holding the **version number the Master currently holds**
- Master cannot ask CSes directly at query time (a CS might be offline → stale answer)
- Instead, master compares stored version numbers against what each CS reported at startup

**Does master update version first, then tell CSes — or tell CSes first, then update?**
→ Master **tells CS first, then updates version #**
→ CS stores the new version number

### Step 2 — Grant a Lease

- Master picks Primary P; all others are Secondary S
- Master **increments the version number**
- Tells P and S the new version number -> P and S store it
- Grants P a **lease** (time-bounded authority to be primary, by default 60s)

**Why leases? — Preventing Split Brain**
> We don't want to falsely designate two CS instances as Primary simultaneously.
> The Master gives P a lease, and both P and M track its expiry.
> When the lease expires → no more client requests processed by that P.
> M can then safely assign a new P without risk of two simultaneous Primaries.

### Step 3 — Data Flow (Client -> P and S)

- Client now knows which CS is Primary
- Client sends a **copy of data to P and all S** — they write to a **temporary location**
- After receiving confirmation from P and S, client tells P to **write the record to the specific offset of the chunk**. And P then asks S to re-order if there were more than one clients writing to same chunk.

### Step 4 — Primary Coordinates the Write

- P writes the record at the offset; S also write
- If **all S reply "yes"** to P → P sends **success** to client
- If **some S did not succeed** → P tells client to **re-issue the request**
- Client knows which S did not succeed

### Partial Write Scenario (Inconsistency)

```
         P (i)          S (ii)         S (iii)
       ┌───────┐      ┌───────┐      ┌───────┐
       │   D   │      │   D   │      │   •   │  ← S(iii) failed for record D
       │   B   │      │   B   │      │   B   │
       │   C   │      │   C   │      │   C   │
       │   B   │      │   B   │      │       │
       │   A   │      │   A   │      │   A   │
       └───────┘      └───────┘      └───────┘
```

Write comes in and offset is at B — even though S(iii) did not succeed for B,
the **file can only be appended**. So the failed slot **remains blank space** (padding).
This is consistent-but-undefined behaviour in GFS terminology.

### Writes to multiple chunks concurrently
Primary will determine the order as always.
E.g: X and Y are two chunks. CS are A, B, C
A is primary for X, C is primary for Y.
Lets say A and B receives two versions for X, Y in this order X1=London, Y1=Carolina followed by X2=Notingham and Y2=Texas

And C receives X2=Notingham, Y2=Texas followed by X1=London, Y1=Carolina. Then for X chunk, it will be X2 that will be made permanent on all A,B,C. For Y chunk, it will be Y1 that will be made permanent. As C is the Primary for chunk Y.

**Atomic Record Append** GFS lets an operation to append data at the end of a file without worrying about your data being overwrriten by any cocurrent writes to get a lock.

**How Appending Works**
Client pushes data to replicas of last file chunk. If data fits in chunk, apply normal write and return. If it doesn't fit pad the chunk to full size and tell client to retry. Any operations should be idempotent.

**Implications for clients**
Prefer appends over writes. No need for locking and readers handle padding and duplicates. Clients can use checksums to cinfirm data validity. Also generate idempotency keys from duplicates. If making multi-chunk writes, writes should take checkpoints so if writes goes down then we don't start from start.

**Replication and how CS are choosen**
GFS keeps 3 replicas of each chunk by default. Rack aware replication to avoid correlation of replica failures while keeping them close.  
Data is lost when all replicas of data are lost.  
Choose replicas with low disk usage and not many recent files.  

**Performance Optimizations**

Master keeps track of replicas for each chunk by heartbeats to CS. If CS fails then it will be replaced by Master with new replica. Data will be copied to new replica and it will be limited to avoid overloading the GFS.

How do we detect **corrupted chunk**? Use checksums for every 64KB of data in memory and stored in Write-Ahead-Log. Its faster than comparing with other nodes. CS verify this everytime a read comes.

> 1. Copy on write file snapshotting

Clients can copy a file with another name: but master is going to add another file pointing to same exact chunks (Something like soft link in linux). Super inexpensive. Primary replica lease is revoked for chunks, clients must ask from master to write. Only when client wants to write to it, then we will duplicate the chunks. This duplication will happen local to the CS to avoid network bandwidth.

> 2. Rebalancing chunks

Master will have to rebalance by moving some chunks around when there is unequal disk usage on CS.

> 3. Master namesapce concurrency

We want to run the master in multi-threaded manner. So each path of file is reader/writer lock. E.g to create a file apple.txt at /fruits path, I will grab a Writer lock at `/fruits/apple.txt` + get a Reader lock at `/` and `/fruits` paths.

> 4. Lazy garbage collection

We want to free up space from chunks that are CORRUPTED(using checksums) or STALE(using version#) or DELETED(using reference counts and look for orphaned chunks). In heartbeats M can ask the CS to delete it.

> 5. Shadow masters

Even if the master goes off. We replicate operation log to shadow master asynchronously. So shadow master can still be used to query for metadata. This metadata maybe a little stale but data will be up to date because chunk data will still be upto date. This is GFS in HA mode. 

### Design decisions made by GFS
- Failure is expected - Operation log and checkpoints on master
- persistent versions to protect against stale replicas, shadow masters, checksums, rack aware replicas
- Fvouring small appends over large writes, then reading many times
- Optimize for network bandwidth by data transfer to all replicas and then data ordering according to primary
- A single master is okay, allows global view of state to do rebalancing chunks easier.

## Strong Consistency — What It Would Require

To achieve strong consistency in GFS:
- **Detect duplicate requests** so that partial writes do not succeed on re-issue
- **Need multiple phases of requests** — 2-phase commit
- If P fails, the new P might not have the latest operation issued by the old P to secondaries

GFS deliberately chose **not** to implement this — complexity vs. use-case trade-off.

---

## Limitations of GFS

1. **Single master bottleneck** — only a limited number of chunks and CS metadata it can store
2. **Limited read/write throughput** if too many clients hit the master simultaneously
3. **No automatic failover/recovery for the master** — master is a single point of failure

---

## GFS received a lot of success

GFS was hugely successful inside Google. The paper influenced essentially every distributed
file system and object store that came after it (HDFS, S3 internal design, Colossus).
The limitations above led Google to build **Colossus** as GFS's successor — with a
distributed master.

---
