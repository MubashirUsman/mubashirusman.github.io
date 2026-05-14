---
layout: post
title:  "Paper: Dynamo: Amazon's Highly Available key-value Store (2007)"
date:   2026-05-09 00:00:00 +0000
categories: distributed-systems
---

These are my notes from reading Dynamo paper, all ideas from the original paper.

At Amazon most applications needed simple read and write operations such as _add to cart_, _best seller lists_,  _customer's wishlist_ etc services, at this stage an RDBMS was an overkill from the perspective of complex query model and far from read/write latency requirements. Most applications stored and retrieved data using a primary key and did not needed ACID guarantees. Additionally high availability and performance are more important than the strong consistency for many Amazon's services. Dynamo has been the underlying storage for the core services of Amazon and has served _tens of millions of requests_ per day. Dynamo has shown that an eventually-consistent system can be used in production with demanding applications.

### Assumptions and Requirements
- Query Model: simple read and write operations to a _data item that is uniquely identified by a key_. Dynamo targets applications that need to store objects that are relatively small(1MB).
- Experience has shown that when a database provides ACID guarantees, it tends to have poor availability. Dynamo targets applications that operate with weaker consistency (the “C” in ACID) if this results in high availability. 
- Dynamo should be efficient, Amazon has stringent latency requirements which are in general measured at _99.9th the percentile_. SLA is based on Percentiles rather than median or averages. E.g a service provides a response within 300ms for 99.9% of requests for a peak client load of 500 requests/sec.
- Another assumption is that _environment is assumed to be non-hostile_ and there are no security related requirements such as authentication and authorization.

### Design Considerations
Tradtionally data replication algorithms perform synchronous replica coordination in order to provide a strongly consistent data access interface, in order to do this they are forced to give up availability under certain failures. It is well known that when dealing with the possibility of network failures, strong consistency and high data availability cannot be achieved simultaneously. To achieve high availability, Dynamo uses optimistic replication techniques, where writes are allowed to happen even during network partition. The challenge is that it leads to conflicting changes which needs to be resolved afterwards.
- First consideration is to to decide when to perform the process of resolving update conflicts, i.e., whether conflicts should be resolved during reads or writes. For Amazon services, rejecting customer updates could result in a poor customer experience. So we want to allow add/remove items from their carts during partitions. This requirement pushes the conflict resolution to reads.
- The next choice is _who_ performs the conlfict resolution, this can be done by the Dynamo or the application. Dynamo can use simple policy such as **last write wins** or use **vector clock**.
- Symmetry: Every node in Dynamo should have the same set of responsibilities as its peers. No nodes with special responsibilities.
- Dynamo prefers decentralised design over centralized control. In the past, centralized control has resulted in outages and the goal is to avoid it as much as possible.
- The system needs to be able to exploit heterogeneity in the infrastructure it runs on. e.g. the work distribution must be proportional to the capabilities of the
individual servers.

## Related Work

Antiquity is a wide-area distributed storage system designed to handle multiple server failures. It uses a secure log to preserve data integrity, replicates each log on multiple servers for durability, and uses Byzantine fault tolerance protocols to ensure data consistency. In contrast to Antiquity, Dynamo does **not** focus on the problem of **data integrity and security** and is built for a trusted environment. Bigtable is a distributed storage system for managing structured data. It maintains a sparse, **multi-dimensional sorted map** and allows applications to access their data using multiple attributes, Compared to Bigtable, Dynamo targets applications that require only key/value access with primary focus on high availability where updates are not rejected even in the wake of network partitions or server failures.

Dynamo is targeted mainly at applications that need an “always writeable” data store where updates are not rejected during failures or conccurent writes. Dynamo is built for latency sensitive applications that require at least 99.9% of read and write operations to be performed within a few hundred milliseconds.

## System Architecture

![Summary of Techniques used by Dynamo](/assets/Dynamo-summary.png)

### System Interface
Dynamo stores objects associated with a key through simple interface using two operations, **_get() and put()_**. The get(key) locates the object or list of objects with conflicts along with the _context_. The put(key, context, object) determines where the replicas of the _object_ are to be placed based on the _key_. The context includes the metadata of the object, inlcuding version of the object. Dynamo applies a MD5 hash on the key to generate a 128-bit identifier, which is used to determine the storage nodes that are responsible for serving the key.

### Partitioning Algorithm

Dynamo’s partitioning relies on _**consistent hashing**_ to distribute the load across multiple storage hosts. It allows to dynamically partition the data. The principle advantage of consistent hashing is that departure or arrival of a node only affects its immediate neighbors and other nodes remain unaffected.

> In consistent hashing, the output range of a hash function is treated as a **fixed circular space** or “ring” (i.e. the largest hash value wraps around to the smallest hash value). Each node in the system is assigned a random value within this space which represents its “position” on the ring. Each data item identified by a key is assigned to a node by hashing the data item’s key to yield its position on the ring. and then walking the ring clockwise to find the first node with a position larger than the item’s position.

As is consistent hashing has some challenges. First, the random position assignment of each node on the ring leads to non-uniform data and load distribution. Second, the basic algorithm is oblivious to the heterogeneity in the performance of nodes. 
- To address first, Dynamo uses the concept of virtual nodes, instead of mapping a node to a single point in the circle, each node gets assigned to multiple points in the ring. Thus if a node becomes unavailable, the load handled by this node is evenly distributed evenly across the nodes.
- For the second problem, when a new node is added to
the system, it is assigned multiple positions (henceforth, “tokens”) in the ring. The number of virtual nodes that a node is responsible can
decided based on its capacity. 

### Replication

Each data item is _**replicated at N hosts**_. Each key is assigned to a co-ordinator node, The **coordinator** is in charge of the replication of the data items that fall within its range. In addition to locally storing each key within its range, the coordinator replicates these keys at the N-1 clockwise successor nodes in the ring. 

The list of nodes that is responsible for storing a particular key is called the _preference list_. Every node in the system knows which nodes should be in this list for a particular key. To account for failures, perference list contains more than N nodes.

### Data Versioning

Dynamo offers eventual consistency, that allows for updates to be propagated to all replicas asynchronously. A put() may return to the caller before the update is propagated to all the replicas and a subsequent get() might return stale result.

If the latest state of the cart is unavailable, then add to cart operation still succeeds even when the latest version is not available and the item is added to (or removed from) the older version and the divergent versions are reconciled later. 

Dynamo treats each version of modification as as a new immutable object. It allows for multiple versions of an object to be present in the system at the same time. Most of the time, new versions subsume the previous version(s), and the system itself can determine the authoritative version. In the presence of failures combined with concurrent updates, results in conflicting updates and in such cases client performs reconciliation. Updates will never be rejected, but some deleted items might re-appear.

Dyanmo uses vector clocks to capture causality between different versions of the object.
> **A vector clock is effectively a list of (node, counter) pairs**. One vector clock is associated with every version of every object. One can determine whether two versions of an object are on parallel branches or have a causal ordering, by examine their vector clocks. If the counters on the first object’s clock are less-than-or-equal to all of the nodes in the second clock, then the first is an ancestor of the second and can be forgotten.

When a client wants to update an object, it must specify which version it wants to update. This version info is taken from the context it obtained from earlier read operation. Upon processing a read request, if Dynamo has multiple versions that it can not reconcile, then it will return all the objects with the corresponding version to the client. Simply put, if a node does not know the relation between two versions, then it will return both to client.

A possible issue with vector clocks is that **the size of vector clocks may grow** if many servers coordinate the writes to an object. In practice, its not a problem as the writes are handled by the top nodes in the preference list. But if the size of the vector clock grows in some scenarios, Dynamo employs the following clock truncation scheme:

Along with each (node, counter) pair, Dynamo stores a timestamp that indicates the last time the node updated the data item. When the number of (node, counter) pairs reaches a threshold (say 10), _**the oldest pair is removed**_ from the clock.

### Execution of get() and put() operations

A client can invoke get/put operations using either: (1) route its request through a generic load balancer that will select a node based on load information (2) use a partition-aware client library that routes requests directly to the appropriate coordinator nodes. The advantage of first is that client does not need to embed any code related to Dynamo in its application. If the request lands at a node that is not the co-ordinator for the key then it must send the request to responsible node. And the advantage of second strategy is that it can lower the latency because it skips the potential forwarding step.

Read and write operations involve the first N healthy nodes in the preference list, skipping over those that are down or inaccessible. **_To maintain consistency among its replicas, Dynamo uses a consistency protocol similar to those used in quorum systems_**. **sloppy quorums**.

> This protocol has two key configurable values: R and W. R is the minimum number of nodes that must participate in a successful read operation. W is the minimum number of nodes that must participate in a successful write operation. Setting R and W such that R + W > N yields a quorum-like system. In this model, the latency of a get (or put) operation is dictated by the slowest of the R (or W) replicas. For this reason, R and W are usually configured to be less than N, to provide better latency.

- Write operation: for a get() request, the coordinator requests all existing versions of data for that key from the N highest-ranked reachable nodes in the preference list for that key, and then waits for R responses before returning the result to the client.

### Handling temporary failures: Hinted Handoff

Dynamo does not use strict quorum membership (nodes in quorum set can change), it uses a **_“sloppy quorum”_**; all read and
write operations are performed on the first N healthy nodes from the preference list. These first N might or might not be the nodes which took part in previous write. 
![hinted-handoff](/assets/hinted-handoff.png)

In this example, if node A is temporarily down or unreachable during a write operation then a replica that would
normally have lived on A will now be sent to node D. This is done to maintain the desired availability.
> The replica sent to D will have a hint in its metadata that suggests which node was the intended recipient of the replica (in this case A). Upon detecting that A has recovered, D will attempt to deliver the replica to A. Once the transfer succeeds, D may delete the object from its local store. Using hinted handoff, Dynamo ensures that the read and write operations are not failed due to temporary node or network failures.

### Handling permanent failures: Replica Synchronisation

To handle scenarios when hinted replicas become unavailable it implements replica synchronisation technique. To detect the inconsistencies between replicas faster and to minimize the amount of transferred data, Dynamo uses Merkle trees. 

> A Merkle tree is a hash tree where leaves are hashes of the values of individual keys. **_Parent nodes higher in the tree are hashes of their respective children_**. The principal advantage of Merkle tree is that **_each branch of the tree can be checked independently_** without requiring nodes to download the entire tree or the entire data set. Moreover, Merkle trees help in **_reducing the amount of data_** that needs to be transferred while checking for inconsistencies among replicas. 

Two nodes may exchange the hash values of children and the process continues until it reaches the leaves of the trees, at which point the hosts can identify the keys that are “out of sync”.

In Dynamo: each node maintains a separate Merkle tree for each key range. Two nodes exchange the root of the Merkle tree
corresponding to the key ranges that they host in common. The disadvantage with this scheme is that many key ranges change when a node joins or leaves the system thereby requiring the tree(s) to be recalculated.

### Membership and Failure Detection

#### Ring Membership

> A gossip-based protocol propagates membership changes and maintains an eventually consistent view of membership. Each node contacts a peer chosen at random every second and the two nodes efficiently reconcile their persisted membership change histories.

When a node joins, it chooses its tokens and maps nodes to their token sets. Initially its just localnode and its tokens. The mappings stored at different Dynamo nodes are reconciled during the same communication exchange that reconciles the membership change histories. Therefore each storage node is aware of token ranges its peer holds. Because of this, any node can forward key/value to the responsible nodes directly.

#### External Discovery

The above mechanism could introduce temporary logically partitioned dynamo ring. For example, if an administrator adds node A to the ring, and then adds node B to the ring, then neither of these nodes will immediately be aware of each other. To prevent this, some Dynamo nodes play the role of seeds. Seeds are nodes that are discovered via an external mechanism and are known to all nodes. Seeds can be obtained from static configuration. 

#### Failure Detection

> Early designs of Dynamo used a _**decentralized failure detector(gossip)**_ to maintain a globally consistent view of failure state. Later it was determined that the explicit node join and leave methods obviates the need for a global view of failure state.

For the purpose of avoiding failed attempts at communication, a _**purely local notion of failure detection**_ is entirely sufficient: Node A may consider node B failed if node B does not respond to node A’s messages (even if B is responsive to node C's messages).

In the presence of a steady rate of client requests generating internode communication in the Dynamo ring, a node A quickly discovers that a node B is unresponsive when B fails to respond to a message; Node A then uses alternate nodes to service requests that map to B's partitions; in the absence of traffic between A and B, there is no need to know whether the other is reachable or unresponsive. 

### Adding/Removing Storage Nodes

When a new node (say X) is added into the system (say between A and B), it gets assigned a number of tokens that are randomly scattered on the ring. For every key range that is assigned to node X, there may be a number of nodes (less than or equal to N) that are currently in charge of handling keys ( in figure 2, B, C and D) that fall within its token range. Due to the allocation of key ranges to X, some existing nodes (B,C,D can offer keys to X) no longer have to some of their keys and these nodes transfer those keys to X. 

## Implementation

In Dynamo, each storage node has three main software components: request coordination, membership and failure detection, and a local persistence engine. Dynamo’s local persistence component allows for different storage engines to be plugged in. Engines that are in use are Berkeley Database (BDB) Transactional Data Store2, BDB Java Edition, MySQL. The main reason for designing a pluggable persistence component is to choose the storage engine best suited for an application’s access patterns. 

---
There are many details which are not covered from EXPERIENCES & LESSONS LEARNED.
