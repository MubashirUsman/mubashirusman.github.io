---
layout: post
title:  "Notes on System Design from First Principles (8-10)"
date:   2026-04-01 13:09:00 +0000
categories: distributed-systems
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

