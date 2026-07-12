---
layout: post
title:  "Notes on System Design from First Principles (8-10)"
date:   2026-04-01 13:09:00 +0000
categories: distributed-systems system-design
---

## Part 8 CAP Theorem — A Deeper Dive

> **Core principle:** With every optimisation, we are making a trade-off.
> A sharded database solves the capacity problem but creates a truth problem. There is no free lunch.

### CAP Theorem Definition

| Property | Precise meaning |
|---|---|
| **C — Consistency** | Every read receives the most recent write. No stale data, ever. |
| **A — Availability** | The system never says "no". No timeouts. Every request gets *some* response. |
| **P — Partition Tolerance** | The system keeps operating despite network failures between nodes. |

> **P is not optional.** Networks fail. Partition tolerance is always present in any real
> distributed system. The real choice is always **C vs A** when a partition occurs.

#### What Actually Causes a Partition?

A partition is not just a broken cable. It happens any time **delays exceed timeouts**:

```
node slow  =  node dead  (from the perspective of other nodes)
```

- CPU freezes for 2 seconds because the **JVM garbage collector** pauses all threads
- **Network buffer fills up** — packets are dropped, not delivered
- A switch fails, a rack loses power, a misconfigured firewall

> If `delay > timeout threshold` → the system declares a partition and must choose C or A.

#### CP Systems — Prioritise safety (truth)

**Use for:** stocks, banks, inventory systems — anywhere data correctness is non-negotiable.

**How it works** CP systems use a **distributed lock / consensus** mechanism (e.g. **Raft**, Paxos):

```
Write request arrives
        │
        ▼
Primary proposes write to all nodes
        │
        ▼
Wait for MAJORITY (Quorum) to confirm   ← e.g. 3 out of 5 nodes must say "yes"
        │
   3/5 say yes?          2/5 say yes?
        │                      │
        ▼                      ▼
  Write succeeds          System STOPS
  (consistent)            (unavailable — rather wrong than incorrect)
```

**During a partition:** "I am temporarily unavailable — I cannot reach the other node."
Availability is sacrificed. Consistency is preserved.

**Trade-off:** CP systems cannot scale to global social media size. Waiting for quorum
on every write is too slow at billions of requests per second.


#### AP Systems — Prioritise Speed (liveness)

**Use for:** Netflix, Twitter, Facebook — anywhere latency directly destroys user experience.

**How it works** Writes are accepted locally and replicated in the background (asynchronous). The system is always up, always fast.

**During a partition:** Nodes keep accepting reads and writes independently.
When the partition heals, they reconcile (using LWW, vector clocks, or CRDTs — see Lecture 7).

**Consequence:** Some users might not see the same data as others at the same instant —
e.g. one user sees a photo liked, another doesn't yet. **This does not matter** for most
social features. Latency mattering more than perfect consistency is the deliberate bet.

### 5. The Split-Brain Problem

> The more you try to be available, the more the risk of **split brain**.

**Split brain** occurs when a network partition causes two nodes to both believe they are
the authoritative leader. Both accept writes. When the partition heals, you have two
diverging versions of truth — and reconciling them may be impossible without data loss.

**CP systems avoid split brain** by refusing to operate without a quorum. You cannot
have two leaders if leadership requires a majority vote.

**AP systems accept split brain** and resolve it after the fact. The question to ask:

> *Is the extra uptime worth the risk of being wrong?*
> **For Twitter → yes. For a bank → no.**

### 6. PACELC in Practice

```
                    DURING partition          NORMAL operation
                  ┌─────────────────┐       ┌──────────────────┐
CP / C system     │  Unavailable    │  →    │  Synchronous     │  →  Consistency (C)
(Postgres, Raft)  │  (won't serve)  │       │  replication     │
                  └─────────────────┘       └──────────────────┘

AP / L system     │  Available      │  →    │  Asynchronous    │  →  Availability / Low
(Cassandra,       │  (stale ok)     │       │  replication     │     Latency (A)
 DynamoDB)        └─────────────────┘       └──────────────────┘
```

### 7. The Decision Framework

Ask two questions about your data:

| Question | Answer | Strategy |
|---|---|---|
| Is this **value data**? | Money, inventory, medical records | → **CP** — prioritise truth |
| Is this **volume data**? | Logs, likes, view counts, analytics | → **AP** — prioritise speed |

Then choose your partition strategy accordingly:

```
Need truth?  →  CP  (consistent, may be unavailable during partition)
Need speed?  →  AP  (always available, eventually consistent)
```

And for normal operations (PACELC — the "else" branch):

```
Synchronous replication  →  Consistency  (C)
Asynchronous replication →  Availability / Low Latency (A)
```

---

## Part 9 Distributed Consensus & Raft

### CAP theorem and Split Brain problem

During a partition, one must choose between being **available** or being **consistent**.

```
         Consistent
            /\
           /  \
          /    \
  Partition ── Available
```

If **truth is paramount** in an unreliable network → choose **Consistency + Partition tolerance**.
Example use case: distributed state machines, leader election, metadata stores.

#### What Causes a Partition?
1. Network gets cut
2. Garbage collection pause freezes a node
3. Network is slow → delays exceed timeouts → node appears dead

**Split brain** happens when different machines give different answers — both believing they are the authoritative leader. There are many ways to achieve strong consistency and avoid split brain, including **2 phase commit**, **3 phase commit** and **Distributed Consensus**.

> **Paxos** is correct but notoriously difficult to understand and implement.
> **Raft** was designed to be understandable and can solve the split brain problem.

#### Why Not Unanimous Agreement?

If we have 5 nodes and we don't proceed with writes until **all 5 agree**, the system becomes very slow — one slow or dead node blocks everyone. And there is zero fault tolerance.

Solution: **Leader-Follower model with quorum majority**.


### Raft — Node States

Every node in a Raft cluster starts as a **Follower** and is always in exactly one of three states:

```
┌──────────────────┐   timeout   ┌──────────────────┐   wins election  ┌──────────────────┐
│    Follower      │ ──────────► │    Candidate     │ ───────────────► │     Leader       │
│                  │             │                  │                  │                  │
│ Passive.         │             │ Trying to        │                  │ Sends heartbeats │
│ Listens, waiting │             │ become a leader  │                  │ Accepts all      │
│ for heartbeats   │             │                  │                  │ writes           │
└──────────────────┘             └──────────────────┘                  └──────────────────┘
        ▲                                                                        │
        └────────────────── sees higher term → steps down ───────────────────────┘

Every node starts as Follower.
```

---

#### Leader Liveness — Heartbeats

**Leader Liveness Broadcasting:**
The leader sends a **heartbeat** at a fixed interval (e.g. ~150 ms) to all followers.
This is like a king shouting *"I am the king"* in the jungle — and all followers are listening.

#### Leader Failure & Election

**Leader Failure — Heartbeat Goes Silent:**

Followers have a **randomised timeout** (150–250 ms). If no heartbeat arrives before timeout:

- Timeouts are **randomised** to avoid a split vote (all candidates voting for themselves simultaneously → chaos)
- The **first follower whose timeout fires** becomes a **Candidate**

**Candidate behaviour:**
- Increments its **term** (logical clock / election year)
- Votes for itself
- Sends `RequestVote` to all nodes in the cluster

**Term** = distributed logical time. It's the election year of the cluster.
By comparing terms, nodes can tell if they are out of date → step down and become a follower.
The term defines who the current leader is.

**Voting Rules:**
A follower votes for a candidate only if **both** conditions are met:
1. It has not already voted in this term
2. The candidate's log is at least as up-to-date as the follower's log

If both conditions hold → follower casts its vote.

**Quorum:**
- Quorum = **majority** = `⌊N/2⌋ + 1`
- When a candidate gets quorum → it becomes the **new leader**

| N (nodes) | Quorum | Fault Tolerance |
|:---------:|:------:|:---------------:|
| 3         | 2      | 1               |
| 4         | 3      | 1               |
| 5         | 3      | 2               |
| 6         | 4      | 2               |

> **Even numbers are slower and more expensive** — they raise the quorum without raising fault tolerance. The **sweet spot is 5 nodes** (tolerates 2 failures, quorum of 3).

#### How Writes Work — The Replicated Log

In Raft, every change is written to a **replicated log**.

```
1. Client sends write to Leader
2. Leader writes entry to its log → marked UNCOMMITTED
3. Leader sends entry to all Followers (AppendEntries)
4. Followers write entry to their log → send ACK back to Leader
5. Once QUORUM of ACKs received → Leader marks entry COMMITTED
6. Leader informs user of success + tells replicas to commit
```

> **The golden path of Raft:** data is only "True" when it has been replicated to a majority.


### RAFT Failure Recovery Scenarios

#### Scenario 1 — Leader Node Goes Down

Example: 5-node cluster, Node 1 is leader, it crashes.

```
Node 3's timer fires first
→ Node 3 increments term, holds election
→ Gets votes from Node 3 (self) + 2 others = quorum (3/5)
→ Node 3 becomes new leader
```

#### Scenario 2 — Network Partition

```
Partition splits cluster:
  Side A: Node 1 (old leader), Node 2
  Side B: Node 3, Node 4, Node 5
```

**What happens to Node 1 (old leader on minority side)?**
- Write arrives at Node 1
- Node 1 writes to log, sends to all 5 nodes
- Only gets ACKs from Node 2 → never reaches quorum in a time bound
- Node 1 **does not confirm** to user → **consistency is preserved**

**Meanwhile on Side B:**
- Node 3's timer fires → election → becomes new leader (gets 3 votes = quorum)
- Side B continues serving writes normally

**When partition heals:**
- Node 1 compares terms → sees it is out of date (lower term than Node 3)
- Node 1 steps down and becomes a follower
- Node 1's uncommitted log entries are **overwritten** by the new leader's log

#### Five Pillars of Raft

| Pillar | Purpose |
|---|---|
| **Leader Election** | One node is authoritative at a time — all writes go through it |
| **Logical Time (Terms)** | Allows nodes to detect stale leaders and out-of-date state |
| **Heartbeats & Timeouts** | Detect leader failure; randomised to avoid split votes |
| **Quorum** | Majority agreement required — tolerates minority failures |
| **Replicated Log** | Ordered, durable record of all state changes across the cluster |

### Consensus Is Slow — Use It Wisely

Consensus has a real cost on every write:
- All writes go through a **single leader**
- **Network hop** to reach quorum
- **Disk sync** (fsync) on each node before ACK

> **Right use of Raft:** metadata, configuration, leader election — low-volume, high-importance data.
> For video analytics or high-throughput data → use an **eventually consistent** system instead.


### Coordination Services — Real-World Raft

For Raft, we use dedicated **coordination services** rather than implementing it from scratch:

| Service | Protocol | Used by | Purpose |
|---|---|---|---|
| **etcd** | Raft | Kubernetes | Stores all pod/deployment/ConfigMap/node state |
| **ZooKeeper** | ZAB | Kafka | Manages which Kafka broker is leader for which partition |
| **Consul** | Raft | Service mesh | Service discovery, health checks, K-V config |

**ZAB (ZooKeeper Atomic Broadcast):** Similar to Raft — also holds elections and maintains an ordered replicated log.

**Google Spanner:** Globally distributed database where **each shard is a Paxos-controlled group of machines**.

---

## Part 10 Caching

### Why Is an App Slow?

Not network. Not bad queries. It's the **memory hierarchy**.

See Lecture 5 for latency numbers. The hierarchy from fastest to slowest:

```
L1 < L2 < L3 < RAM < SSD < Network
```

The solution is to keep hot data as high up this hierarchy as possible.

#### The 90-10 Rule (Locality of Reference)

In almost all systems ever built, **90% of all requests are for 10% of the data**.

Real examples:
- Tweets from the last few hours are being read constantly
- Only the top videos are being watched 90% of the time
- Famous products on Amazon get the vast majority of traffic

#### Two types of locality

| Type | Meaning |
|---|---|
| **Temporal locality** | If data was accessed recently, it is likely to be accessed again soon |
| **Spatial locality** | Data *near* the accessed data will likely be accessed soon |

> Cache holds the **hot 10%** of data. If cache hit rate is 95%, then only 5% of
> requests ever reach the disk.


#### The Tradeoff with Cache

| Cost | Detail |
|---|---|
| **RAM is expensive** | 10–50× the cost of SSD per GB |
| **Volatile** | Cache is lost on crash — it's in RAM |
| **Inconsistency risk** | Two copies of data exist (cache + DB) — they can diverge |


### Caching Strategies

#### Cache Aside (Lazy Loading)

```
Request arrives
      │
      ▼
Check cache ──► HIT?  ──► return data :)
      │
      ▼ MISS
Go to DB
      │
      ▼
Save result in cache
      │
      ▼
Return data to user
```

**Used by:** Redis with Django, Node.js apps

| Pros | Cons |
|---|---|
| Resilient — only caches what's needed | First request is always a miss |
| Cache failure doesn't break the app | **Stale data risk** — if DB changes, cache is not updated |


#### Write Through Caching

Every write goes through the cache **first**, then persists to DB.

```
Write request → Cache → DB (synchronously)
```

| Pros | Cons |
|---|---|
| Always consistent — cache and DB in sync | Writes are still slow (must wait for DB) |
| No stale reads, fast reads | Wastes cache space on rarely-read data |

**Best for:** pricing data, account info, anything that **must be fresh**.


#### Write Back (Write Behind)

Write arrives → cache returns **"success" instantly** → marks data as **"dirty"** →
flushes to DB later in a **big batch**.

```
Write request → Cache ("success" immediately) ──► [dirty] ──► DB (batch, later)
```

| Pros | Cons |
|---|---|
| **Fastest write speed** — millions of writes/sec | **Data loss** if cache crashes before flush |
| Batching reduces DB load | |


### Cache Invalidation

The hardest problem in caching — **data gets stale**. Three strategies:

#### TTL (Time To Live)
Keep a cache entry for a fixed duration. After TTL expires, entry is deleted.

| Pros | Cons |
|---|---|
| Predictable and simple | Data can be wrong for up to the full TTL window |

> Fine for social media. **Not okay** for banking apps.


#### Eviction Policy
When the cache is full, decide what to remove:

| Policy | Behaviour | Notes |
|---|---|---|
| **LRU** (Least Recently Used) | Evicts oldest unused data | Most common. Fits temporal locality perfectly. **Redis uses this.** |
| **LFU** (Least Frequently Used) | Evicts least frequently accessed data | Less common |

#### Manual Invalidation
App updates DB → sends a notification to cache → cache deletes/updates that entry.

| Pros | Cons |
|---|---|
| Immediate consistency | **Not reliable** — notification can be lost over the network |
| Fine-grained control | Easy in small environments, **very hard at scale** |

> Every invalidation strategy has a disadvantage. There is no perfect answer.


### Cache Failure Modes

#### Thundering Herd Problem
When a cache server crashes, cache is no longer a shield. In this case **all queries go directly to the DB** --> all load falls on the database --> DB may also crash.

**Solution: Cache Warming**
Prepopulate the cache with hot data *before* sending live traffic to it.

#### Cache Penetration
Deliberate (or accidental) requests for **keys that don't exist** in the cache.
Every single query misses the cache and hits the DB.
This can be used as a **DDoS vector** against your database.

**Solution: Bloom Filter**
Put a Bloom filter in front of the cache. If the Bloom filter says the key definitely
doesn't exist ==> reject the request immediately, never touch the DB.


### Caching Is Fractal

Caching doesn't just exist at the application layer — it exists **at every layer** of the stack:

| Layer | Cache |
|---|---|
| Browser | HTML, CSS, JS (Cache-Control headers) |
| OS | Page cache (RAM buffer for disk reads) |
| CDN | Videos, images, static assets at edge |
| DB | Buffer pool (DB's own in-memory page cache) |
| Application | Redis, Memcached |

#### When deciding which layer needs caching

Ask two questions:

```
Is the distance too much?  →  CDN is the solution
Is the query too slow?     →  Redis is the solution
```


#### Conclusion — When to Cache

> **Trade consistency for scale** — that is what a cache offers.

The rules:
- Only add a cache when **a single Postgres instance is not enough**
- Trust the **90-10 rule** — most data is cold, cache the hot tail
- Always **plan for invalidation** before adding a cache
- The job of a cache is to **protect the database**

---
