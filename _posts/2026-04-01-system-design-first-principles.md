---
layout: post
title:  "Notes on System Design from First Principles (1-7)"
date:   2026-04-01 13:09:00 +0000
categories: distributed-systems system-design
---

# System Design from First Principles

## Part 1,2 Physics of data

Data fetch time depends on distance it has to travel, more frequently the data needs to be accessed the close it should be to CPU.

**Little's law** - `L = λW`
- L = average number of requests in the system
- λ = rate of requests coming in
- W = average time it takes in the system, so load on a system is proportional to the time taken to process a request.

Average latency does not give true picture of delays so we use percentiles, such as 99^th^ means the 1 percent of requests that faced highest delay.

For reliability, instead of using uptime it can be defined in terms of successful number of responses with respect to total responses. Uptime can be misleading as for a distributed service, our service will remain available in some parts of the world at all times.

System should be run at less capacity than it can handle, for example 70% CPU usage, the rest should be left for unexpected load spike to protect the system from creating a cascade failure. In fact its a tradeoff between resource wastage and avoiding exponential increase in latency in the face of high traffic.

**Amdahl's law** speed of doing a task is limited by the serial fraction of the task(the one that can not be parallelized).
```
     1
------------
    (1-s)
s + -----
      n
```
Here `s` is the fraction of task to be done serially, and `n` is the number of processors. E.g if code is 95% parallelizable and 5% is serial and n goes to so large that fractional part becomes zero, then (1/0.05) = 20, even if n is infinity, we can not achieve more speed than 20 times.

### Four golden signals
- Latency, time to service a request, we should track P50, P90, P99
- Traffic, the demand of the system or number of requests coming in
- Errors, ratio of failed requests to the total requests
- Saturation, how full is the system, such as database connection pool, cpu usage etc

---

## Part 3 Communication

### Data access latency
This table shows cpu access time for different storage medias.

| Storage   | Scaled Latency| Actual Latency  |
|-----------|---------------|-----------------|
| L1 Cache  | 0.5 seconds   | 1-4 nano sec    |
| RAM       | 2 minutes     | 100+ nano sec   |
| SSD       | 2 days        | 25-100 micro sec|
| HDD       | 5 months      | 5-10 mili sec   |

A network call is expensive due to: **Latency due to network calls**
> (1) it needs to do a dns query (2) needs to do TCP 3-way handshake, (3) TLS handshake (4) calculate encryption keys (5) invisible timeouts 

### Serialization Latency
Serialization is expensive: **Latency due to serialization**
Constructing json objects to be sent on the network from Java objects (serialization) is CPU intensive (requires to do string manipulation). Instead of JSON, we can use other techniques to serialize data.

| JSON                         | Protobuf OR Flat buf                     |
|------------------------------|------------------------------------------|
| {"id": 5, "status": active}  | bytes                                    |
| CPU needs to do json parsing | CPU copies bytes so no parsing is needed |
| Not good for high performance| Good for high performance                |

> flat buffers > protobuffers > json

### Network Latency
Speed of light in fiber optic is a hard limit on data transfer speed, a total distance from London to San Francisco is 8500 KM, and one trip from London to San Francisco would take 85ms, so with HTTP/1.1 and TLS/1.2 we can calculate total round trip time for one http request sent,
TCP Handshake[85ms] + TLS trip[85ms] + HTTP[85ms] = 225ms

> one RTT 10ms per 1000KM

### QUIC
TCP should be used for rare and long lived connections. QUIC (built on UDP) combines TCP and TLS handshake, so network latency from LONDON to San Francisco becomes: TCP Handshake and TLS trip[85ms] + HTTP[85ms] = 170ms. This is 1-RTT instead of 2-RTT in TCP. For returning users this will reduce to 85ms or 0-RTT, using CDN the edge server can keep an open TCP connection with root server and this latency can be further reduced.

### HTTP2
HTTP2 sends all requests using single TCP connection (multiplexing, connection pooling), and if combined with ptotobuf then serialization and deserialization cost can be saved.

Apache Arrow defines standard memory layout, achieves zero copy deserialization when data on network cable, on disk and in the ram is identical.

> Good rules of thumb for API design: **Batching** send one request with many little things, **Data locality** if two services communicate too much, consider making them one, **Coarse grained API** is not too chatty

---

## Part 4 Anatomy of a Request

- URL in browser memory -> syscall connect() -> context switch to TCP/IP stack -> 
- MTU limit on packet is 1500 Bytes and TCP header comes in
- IP address needs to be searched and DNS comes in, Geo-DNS gives IP according to location of user
- Ethernet header
- After leaving ISP, BGP comes in and it decides on which path to choose for the given destination
- Anycast help BGP by announcing same IP from different locations
- Edge server comes in, TLS is terminated, since TCP handshake is expensive so edge server uses existing warm TCP connection to root server
- Firewall for DPI, packet inspection is expensive so keep it as close to edge as possible
- Load balancer, layer 7
- API gateway, does user exist and is JWT token valid, rate limiting, protocol translation REST to gRPC
---

## Part 5 Persistence

### Fundamental challenge
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

> We want persistence *and* speed. These two goals conflict — and the rest of this lecture is about how to reconcile them.

---

### Databases & The OS Lie

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

### Write-Ahead Log (WAL)

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

### SSD Internals — Write Amplification

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

### B-Tree — Shallow and Fat

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

### LSM Tree — Log-Structured Merge Tree

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

### The RUM Conjecture

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

### The Invisible Enemy — Bit Rot

Even at rest, data can silently corrupt. Cosmic rays, voltage fluctuations, and magnetic interference can flip bits without the OS noticing. **Do not trust hardware.**

**Solution:** Checksums (e.g. SHA-256). Compute a hash on write; recompute and verify on read. ZFS does this automatically for every block.

#### Disk Failure Rates

Out of 10,000 disks, approximately **1 fails per month**. At scale, disk failures are expected daily events — not exceptional ones. Distributed systems must treat failure as the norm.

#### 2003 — Google File System (GFS)

GFS splits data into chunks, storing **3 copies across 3 machines on 3 different racks**. If one rack goes down, data survives on the other two.

But this introduces a **consistency problem**: all three replicas might have slightly different data at any moment. Consensus protocols like **Raft** solve this.

---

### The Big Trade-Off Spectrum

Every persistence decision sits on a spectrum between maximum speed and maximum durability:

```
   |--------------------------------|---------------------------------|
High risk / High speed              ### 1.                    Low risk / Low speed

  RAM /                              fsync()            Distributed replication S3 /
Buffered I/O                           SSD                 Multi-region
```

#### Practical Decision Framework

**Is it a like on a TikTok post?**
- It's okay if people see the count update with a 2–3 second delay.
- Use an **LSM tree** with buffered I/O. Optimise for throughput.

**Is it a bank transfer?**
- Data loss is unacceptable.
- Use **`fsync()`** and wait for *all* replicas to reply before returning success. Optimise for durability.

---

## Part 6: How to Choose a Database

### The Golden Rule

> **Choose a database based on how you will *access* the data later, not how it looks.**

Audit your **access patterns** first. The shape of your queries determines the right database — not the shape of your data.

### Relational Databases (SQL)

**Core idea:** Data is stored in tables and can be queried with flexibility. 

- Store every fact exactly once. This is the concept of **normalisation**.
- ACID guarantees are provided by relational databases.
- Data is spread across flat tables, and **joins** combines them together at query time.

**The cost of joins**
A join means going to a **different location on disk** for each related row. This is **random I/O** — slow and expensive.

```
Table A (disk location 1) ──JOIN──► Table B (disk location 2)
                                          └──► Table C (disk location 3)
```

> Flat tables in SQL = objects in Java. Normalisation is the mismatch you pay at the database layer to avoid redundancy.

---

### NoSQL — Key-Value Store

**Core idea:** Save a value, get it back by key. That's it.

- Internally a **K-V hash map**
- Lookup is **O(1)**
- You **cannot** query by anything other than the stored key — unlike SQL
- You cannot do aggregations, filters, or joins

```
set("user:42", { name: "Alice", age: 30 })
get("user:42")  ──►  { name: "Alice", age: 30 }
```

**Examples:** Redis, DynamoDB, Memcached

**Use when:** you know the exact key and just need to retrieve a value fast.

---

### NoSQL — Document Database

**Core idea:** Save all related data in one document. Just dump the JSON. Store everything at **one spot on disk**.

- Trades **normalisation** for **data locality**
- No joins needed — all data for an entity lives together
- Downside: leads to **redundancy** (same data duplicated across documents)

```json
{
  "user_id": 42,
  "name": "Alice",
  "orders": [
    { "item": "Book", "price": 12.99 },
    { "item": "Pen",  "price": 1.49 }
  ]
}
```

**Examples:** MongoDB, CouchDB

| Use when | Avoid when |
|----------|------------|
| Data is **self-contained** | Data is **interconnected** |
| You read one entity at a time | You need cross-document queries |
| Schema evolves frequently | Strong consistency is required |

---

### NoSQL — Graph Database

**Core idea:** Relationships are **first-class citizens**. Data is stored using pointers between nodes.

- In SQL, relationships are implicit (foreign keys, joins)
- In a graph DB, the relationship edge is an actual stored object with properties
- Traversing connections is cheap — no joins, just pointer-following

```
(Alice) ──[FOLLOWS]──► (Bob) ──[FOLLOWS]──► (Carol)
  └──────[LIKES]──────► (Post #7)
```

**Examples:** Neo4J

**Use when:** your access pattern is "find all friends of friends" or "what path connects A to B?" — i.e., highly connected, graph-shaped data.

---

### NewSQL — Best of Both Worlds

**Examples:** Google Spanner, CockroachDB

NewSQL databases give you:
- SQL **ACID interface** (familiar query model)
- ACID **guarantees** (correctness)
- **Horizontal scaling** (scale out like NoSQL)

They use a **consensus algorithm** (e.g. Paxos, Raft) to agree on writes across nodes.

> **Trade-off:** You trade individual write speed for horizontal scale. Each write is slower because it must be agreed upon by multiple nodes.

---

### The Decision Framework — Audit Your Access Pattern

| Query type | DB model | Example |
|---|---|---|
| Give me this exact id | **Key-Value** | Redis |
| Summary / aggregation | **Relational** | Postgres |
| Self-contained entity | **Document** | MongoDB |
| Find similar items / embeddings | **Vector** | Pinecone, pgvector |
| Social graph / connections | **Graph** | Neo4J |

#### Why Postgres is often the default

Postgres alone supports all three of:
- **Index lookup** → behaves like a K-V store
- **JSON columns** → behaves like a document store
- **Full SQL model** → relational queries, aggregations, joins

Unless your access pattern is extreme (massive write throughput, purely graph-shaped data), Postgres handles it.

---

### Schema on Write vs. Schema on Read

```
Schema on Write (SQL)          │  Schema on Read (NoSQL)
───────────────────────────────│────────────────────────────────
Define schema first,           │  Start writing immediately,
then write data                │  define schema in application code
                               │
Enforced at the DB layer       │  Enforced at the application layer
                               │
Migration required to          │  Old + new field names must both
change a field name            │  be supported in code
```

> **The schema always exists.** In SQL it lives in the database. In NoSQL it lives in your code. If you change a field name in NoSQL, your codebase must now handle both the old and new name simultaneously — in every service that reads that data.

Flexibility in NoSQL is real, but it comes with a hidden cost that compounds over time.

---

### Multi-Model Databases

**Examples:** Azure Cosmos DB, ArangoDB

These let you store data in **multiple models within the same database** — relational, document, and graph, all in one system. Useful when a single product needs several access patterns without managing multiple separate databases.

---

### Vector Databases

**Use case:** "Find me photos similar to this one."

Vector DBs store **embeddings** (high-dimensional numerical representations of data) and let you search by **similarity** rather than exact match. This is the engine behind recommendation systems, semantic search, and image search.

---

> The key is that every model has a tax. Each database model optimises for something — and pays a price somewhere else:

| Model | What you gain | What you pay |
|---|---|---|
| **Relational** | Flexibility, normalisation | **Join tax** (random I/O) |
| **Key-Value** | O(1) lookup speed | **Query tax** (no filtering, no aggregation) |
| **Document** | Data locality, no joins | **Relational tax** (redundancy, no cross-doc queries) |
| **Graph** | Cheap relationship traversal | **Search tax** (poor at non-graph queries) |

---

### Polyglot Persistence

Large systems often use **multiple databases** — one for each access pattern. This is called **polyglot persistence**.

**Example: Netflix**

| Data | Database | Reason |
|---|---|---|
| User profiles | **Postgres** | Relational, ACID, flexible queries |
| View history | **Cassandra** | High write throughput (LSM tree) |
| Recommendations | **Redis** | Fast K-V cache, sub-millisecond reads |

> Different parts of the same product have different access patterns. Use the right tool for each.

---

## Part 7: The World of Sharding

**Why Shard at All? — Physics Sets the Limits**

Software abstraction can feel like an infinite universe, but physics imposes hard limits:
**speed of light, temperature, IOPS, OOM (out of memory)**. A single machine will always
have a ceiling.

Before sharding, exhaust simpler options in order:

| Option | Approach |
|---|---|
| 1. Optimise queries | Indexes, query rewrites, avoid N+1 |
| 2. Delete old data | Archive or TTL old rows |
| 3. Add a caching layer | Redis in front of the DB |
| 4. Vertical scaling | Put it on a bigger machine |
| 5. **Shard** | Only when the above fail |

> **Conclusion:** The best way is to **not shard**. Sharding loses simplicity, loses joins,
> and loses ACID transactions. It is the last resort.

### Sharding — Horizontal Scaling

**Sharding** = split data into pieces across multiple servers. A **router** takes a
**shard key** and routes the request to the correct server.

Good shard key candidates: `user_id`, `tenant_id`, `europe_user`, `year_num`

#### a. Range-Based Sharding — O(1) lookup

```
users 1      → 25,000   ──► Server 1
users 25,001 → 50,000   ──► Server 2
```

**Problem:** Data is not uniform. All January data goes to Shard 1, all February data
goes to Shard 2. If writes are time-based, a single shard receives *all* current writes.
This is a **hot spot**.

#### b. Hash-Based Sharding

```
id  ──►  hash(id)  ──►  hash(id) % N  ──►  Shard A / B / C
         (md5, crc32)     modulo
```

**Advantages:**
- Even sequential IDs are spread across different/random servers because of hashing
- Guarantees even distribution
- No hot spots - no single server gets all the load

**Problem — Resharding storm:** When you add a server, `N` changes. Almost **(N-1)/N**
of all keys need to move to a new location. Network bandwidth fills, CPU spikes,
I/O is saturated. This is the **resharding storm**.


### Consistent Hashing — Solving the Resharding Storm

Developed at MIT. Used by **DynamoDB, Cassandra, Discord**. We map keys and servers on the hash circle output, so we are already making sure that all servers and keys are distributed in the equaivalent fashion.

Imagine the output of a hash function (e.g. SHA1 → 160-bit integer) as a **ring** (circle).

**How it works**

**Step 1 — Map servers to the ring:**
```
Server A  →  12 o'clock
Server B  →   4 o'clock
Server C  →   8 o'clock
```

**Step 2 — Place data:** Hash the data's ID to get a point on the ring.
e.g. `id 15` hashes to `2 o'clock`.

**Step 3 — Find the owner:** Walk **clockwise** from the data's point until you hit a server.
`2 o'clock` → walk clockwise → hit **Server B** at 4 o'clock. Server B owns this data.

**Step 4 — Adding a new node:** Hash the new server → it lands between two existing servers.
Only the data between the new server and its predecessor needs to move.

> **Key property:** Adding 1 server to an N-server ring moves only **1/N** of data.
> Compare to hash-based sharding where **(N-1)/N** of data moves.

#### Virtual Nodes — Solving Uneven Distribution

Servers may not land uniformly on the ring by chance.

**Solution:** Pretend each physical server is **100 virtual servers**.
- Server A → A1, A2, ... A100 (100 spots on the ring)
- Server B → B1, B2, ... B100

By the **law of large numbers**, these virtual nodes mix evenly across the ring,
guaranteeing uniform distribution even with few physical servers.

### The CAP Theorem

When a **network partition** (communication failure between nodes) occurs, you must
choose between **two** of:

| Property | Meaning |
|---|---|
| **C** — Consistency | Every read returns the most recent write |
| **A** — Availability | Every request gets a response (not guaranteed to be latest) |
| **P** — Partition Tolerance | The system keeps working despite network failures |

> Partition tolerance is non-negotiable in distributed systems. So the real choice is
> **CP vs AP** when a partition happens.

**CP Systems**
- Always correct data
- Strong guarantees
- **Can be unavailable** during a partition
- Slower
- **Examples:** Banks, stock trading, inventory systems, **Postgres**

**AP Systems**
- Always available
- Fast and responsive
- **Temporarily inconsistent** during a partition
- Eventual consistency only
- Partial Quorums
- **Examples:** Caching systems, DNS, social media, **DynamoDB, Cassandra**

---

### PACELC Theorem — Beyond CAP

CAP only describes behaviour *during* a partition. **PACELC** extends it to normal operation:

```
If Partitioned  →  choose: Availability  vs  Consistency
Else (normal)   →  choose: Latency       vs  Consistency
```

#### The Two Replication Strategies

```
DB with Primary + 2 Replicas:
            Primary
           /       \
      Replica 1   Replica 2
```

**If Consistency > Latency (C > L) — Synchronous Replication:**
```
Write arrives → Primary writes → waits for ALL replicas to confirm → tells user "success"
If primary fails and replicas have not returned confirmation, then no problem primary has also not made the write.  
If the primary fails when replicas have already written but primary crashed before hearing the conformation, then primary has not made the write, and client will be stuck in waiting for primary to answer. In this scenario a replica has to be promoted to primary manually(in 2PC) or automatically.

```
- Safe but slow
- Example: **Postgres**
- PACELC profile: CP / C (always consistent, can be slow)

**If Latency > Consistency (L > C) — Asynchronous Replication:**
```
Write arrives → Primary writes locally → tells user "success" → updates replicas in background
```
- Fast but risky (replicas may lag behind)
- Example: **DynamoDB / Cassandra**
- PACELC profile: AP / L (fast normally, inconsistent during partition)


### Consistency Spectrum (Consistency Models)

Weaker consistency = faster system. **Pick the weakest model you can tolerate.**

#### Eventual Consistency
Replicas will *eventually* agree, but reads may return stale data temporarily.

Sub-guarantees you can layer on top:

| Model | Guarantee | Implementation |
|---|---|---|
| **Read-your-writes** | After you write, *you* always see your own write (others may not yet) | Pin user's connection to same replica for a few seconds |
| **Monotonic reads** | Once you've seen version V2, you never see V1 again | Route user's reads to same replica consistently |
| **Causal consistency** | Cause always appears before effect | Track causal dependencies between operations |

---

### Conflict Resolution — When Two Nodes Disagree

When two nodes get updated simultaneously/during partition with different data, you need a conflict resolution strategy to merge:

#### Strategy 1 — Last Write Wins (LWW)
- Simple but dangerous
- Uses timestamps to determine which write is "latest"
- **Cassandra uses this by default**
- **Problem:** Clock skew between servers can silently discard valid writes — data loss with no error

#### Strategy 2 — Vector Clocks (Version Vectors)
- Each node keeps a counter tracking how many times it has seen each version
- Can detect whether updates are **descendant** (one is newer) or **concurrent** (happened independently)
- If concurrent → **store both versions**, surface the conflict to the application
- **DynamoDB uses this** — you may see duplicate items that won't be auto-deleted
- Application code must resolve the conflict

```
Node A: {A:1, B:0}  ──► writes "apple"
Node B: {A:0, B:1}  ──► writes "orange"   (concurrent — neither descends from the other)
                                            → store both, let app decide
```

#### Strategy 3 — CRDTs (Conflict-free Replicated Data Types)
- Mathematical data structures where **all orderings of updates reach the same final state**
- No conflicts possible by design
- Example: you add "apple", someone else adds "orange" → merge = set `{apple, orange}`
- **Figma and Google Docs use CRDTs**
- Best for: counters, sets, text (operational transforms)


### High Availability (HA) — What It Actually Costs

AP systems are obsessed with self-healing and uptime. HA is measured in "nines":

| Availability | Downtime per year |
|---|---|
| 90% | 36.5 days |
| 99% | 3.65 days |
| 99.9% | 8.76 hours |
| 99.99% | 52.56 minutes |
| 99.999% | 5.26 minutes |


### The Celebrity / Hot Key Problem

**Scenario:** Millions of requests arrive for a single key simultaneously.
Example: a tweet from Selena Gomez. All requests hit the same shard → that server melts.

#### Solution — Key Splitting

Append a random suffix to the key. Define a **split factor** of e.g. 10:
```
"tweet:123"  →  "tweet:123_0", "tweet:123_1", ... "tweet:123_9"
```
Writes are spread across 10 different servers.

**Trade-off — Read-Write Amplification:**
- Writes: **10× faster** (spread across servers)
- Reads: **10× slower** (must query all 10 shards, sum results in application)

> Only use key splitting if the system is **write-heavy** and can tolerate slower reads.


### The ID Problem — Globally Unique IDs

Auto-increment IDs break in distributed systems — there is no single central counter.
Two servers can generate the same ID → data corruption.

**Requirements for a distributed ID:**
- Globally unique (no coordination needed between servers)
- Sortable / sequential (B-trees need this to avoid random I/O)
- Embeds time (so B-trees can append data in order)

#### Snowflake ID (Twitter, 2010) — 64-bit integer

```
┌────────┬──────────────────────┬──────────────────┬─────────────────┐
│ 1 bit  │      41 bits         │    10 bits       │    12 bits      │
│ (sign) │    Timestamp (ms)    │   Machine ID     │ Sequence number │
└────────┴──────────────────────┴──────────────────┴─────────────────┘
```

| Component | Detail |
|---|---|
| **Timestamp** | Milliseconds since epoch — makes IDs sortable |
| **Machine ID** | Identifies which server generated the ID — supports **1024 shards** |
| **Sequence** | Resets every millisecond — handles bursts of IDs within the same ms |

**Properties:**
- **Sortable** → B-tree can append data efficiently (no random I/O)
- **69 years** of unique IDs before timestamp overflows
- **No coordination** needed between servers

> Machine IDs must be pre-assigned differently for each server to avoid collisions.

---

### Zero-Downtime Migration Playbook

When migrating from one database (or schema) to another, you cannot go offline.
The 5-stage process:

#### Stage 1 — Dual Writes
Modify application code to write to **both** old and new databases.
- New DB writes are **best-effort** (they do not block the user's request)
- All new data starts flowing to the new sharded cluster

#### Stage 2 — Backfill
Write a script to copy historical data from old DB to new DB.

**Race condition risk:** A row updates in the old DB *after* your script already copied it →
old data overwrites newer data in new DB.

**Fix:** Make writes to new DB **conditional** — only update a row if the incoming
timestamp is **newer** than what is already stored. Never overwrite with stale data.

#### Stage 3 — Verify
Continuously pick **random rows** from both DBs and compare them.
**Do not proceed until consistency is 100%.**

#### Stage 4 — Switch Reads
Flip application reads to the new DB.
**Keep writing to both DBs** — the new DB may be misconfigured for unknown reasons.
Writes still go to both places as a safety net.

#### Stage 5 — Switch Writes
If everything is stable for ~1 week, remove the dual-write code.
Migration complete.

```
Dual write  ──►  Backfill  ──►  Verify  ──►  Switch read  ──►  Switch write
```

---
