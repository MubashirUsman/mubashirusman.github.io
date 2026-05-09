---
layout: post
title:  "Paper: Dynamo: Amazon's Highly Available key-value Store (2007)"
date:   2026-05-09 00:00:00 +0000
categories: distributed-systems
---

At Amazon most applications needed simple read and write operations such as _add to cart_ or _customer's wishlist_ etc, at this stage an RDBMS was an overkill from the perspective of query model and far from read/write latency requirements. Most applications stored/retrieved data using a primary key and did not needed ACID guarantees.

