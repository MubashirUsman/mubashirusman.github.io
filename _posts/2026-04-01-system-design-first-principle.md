---
layout: post
title:  "Notes on System Design from First Principles"
date:   2026-04-01 13:09:00 +0000
categories: distributed-systems
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
TCP should be used for rare and long lived connections. QUIC (built on UDP) combines TCP and TLS handshake, so network latency from LONDON to San Francisco becomes: TCP Handshake and TLS trip[85ms] + HTTP[85ms] = 170ms. For returning users this will reduce to 85ms or 0-RTT, using CDN the edge server can keep an open TCP connection with root server and this latency can be further reduced.

### HTTP2
HTTP2 sends all requests using single TCP connection (connection pooling), and if combined with ptotobuf then serialization and deserialization cost can be saved.

Apache Arrow defines standard memory layout, achieves zero copy deserialization when data on network cable, on disk and in the ram is identical.

> Good rules of thumb: **Batching** send one request with many little things, **Data locality** if two services communicate too much, consider making them one, **Coarse grained API** is not too chatty
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
High risk / High speed                                 Low risk / Low speed

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
