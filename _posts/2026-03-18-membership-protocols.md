---
layout: post
title:  "About SWIM protocol"
date:   2026-03-07 13:30:00 +0000
categories: distributed-systems system-design
---

SWIM (Scalable, Weakly-Consistent, Infection-Style, Processes Group Membership Protocol) is a membership protocol, which is used in distributed systems to answer this: _who are my peers?_ The scaleable in the name implies that it can handle increased size of the system without degrading performance. We build distributed systems in large environments, because scalability is needed. This means thousands of machines could be the in the cluster.  
Gossip protocols work like how people gossip in a society, talking to only few people to share information and then those few people talk to others and then the whole society knows about it. That's hownodes communicate with subset of their total peers to send messages, Infection-Style in the name implies its a gossip protocol.  
Weekly consistent means that after some amount of time, all replicas will agree on the same value, where _some_ is undefined amount of time.  

