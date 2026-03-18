---
layout: post
title:  "About SWIM protocol"
date:   2026-03-07 13:30:00 +0000
categories: distributed-systems system-design
---

SWIM (Scalable, Weakly-Consistent, Infection-Style, Processes Group Membership Protocol) is a membership protocol, which is used in distributed systems to answer this: _who are my peers?_ That means it should do the failure detection and update the peers to only keep the healthy nodes.  
The scaleable in the name implies that it can handle increased size of the system without degrading performance. We build distributed systems in large environments, because scalability is needed. This means thousands of machines could be the in the cluster.  
Gossip protocols work like how people gossip in a society, talking to only few people to share information and then those few people talk to others and then the whole society knows about it. That's hownodes communicate with subset of their total peers to send messages, Infection-Style in the name implies its a gossip protocol.  
Weekly consistent means that after some amount of time, all replicas will agree on the same value, where _some_ is undefined amount of time.  

## Detecting Failure
- `T` is the period time
- `k` is the number of nodes in failure detection group
A node `A` sends a `PING` message to node `B`, if the node replies with `ACK` no further action is needed, but if node does NOT reply before the timeout(less than `T`), then it marks the node `B` suspicious and selects some arbitrary `k` peers and asks them to ping node `B` on behalf of `A`. This is `indirect ping`. If none of `k` nodes receive the `ACK` then that node is marked `dead`. This reduces the number of messages sent to `O(N)` size.

## Information dissenminating
- `JOINED` is sent by a node `P` to inform about the network.
- `FAILED` is sent to peers when a node failure is detected by the above process.
These messages are sene along with the `PING/ACK` to the peers and it results in efficient communication by reducing the size of information dissemination to `O(log(N))`.
