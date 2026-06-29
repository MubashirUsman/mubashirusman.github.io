---
layout: post
title:  "ZooKeeper: Wait-free coordination for Internet-scale systems"
date:   2026-06-27 00:00:00 +0000
categories: distributed-systems
---

## Introduction
Zookeeper was originally created in Yahoo and paper was published in 2010.
Zookeeper is a system that provides co-ordination primitives to solve complex problems such as distributed configuration, group membership, distributed locks, leader election etc. Zookeeper provides FIFO ordering guarantees for the incoming read requests and lineaziability guarantee for requests that change state of zookeeper. _Requests from a single client are processed in the same order they were sent._ Zookeeper allows tens to hundreds of thousands of transactions per second. Workloads often are read heavy with read to write proportion between 2:1 to 100:1. 

1. ZK is wait free, clients can have many outstanding requests which is not yet processed => helps in performance
2. FIFO client ordering guarantee => enables the async operation in the bullet 1
3. Writes are linearisable => updates have a total global order
4. ZK workloads are read heavy, and servers handle reads locally, ZAB is not used to apply total order there.
5. Caching data on clients side improves performance as clients don’t have to probe ZK all the time. ZK uses watch mechanism to notify clients instead of managing cache directly (as in Chubby).
6. Chubby blocks, invalidates cache and to avoid getting blocked by faulty clients indefinitely, it uses leases. But ZK avoids this problem by using watch mechanism.

## Zookeeper Service
ZK provides a hierarchical structure with data nodes called znodes, it resembles to a UNIX filesystem. `znodes` can store data and can be **EPHEMERAL** or **REGULAR** along with being sequential or not. Znodes can have children if they are not ephemeral. Sequential flag adds monotonically increasing number to the name of the znode. 

ZK implements **watches to notify clients if a node data is updated**. ZK only notifies the client and do not send the update, client has to read the node to get the update. This way clients don’t have to do polling. ZK znodes can store application metadata necessary for co-ordination, though they are not meant to be for data storage. 
A client initiates a session by connecting to ZK, session has a timeout. ZK considers the client as faulty if it does not hear anything from client within timeout.

**ZK API** 
create(path, data, flag). 
delete(path, version). 
exists(path, watch)
getData(path, watch). 
sync(path)
All methods have both a synchronous and an asynchronous version available through the API.
Note that ZooKeeper does not use handles to access znodes. Each request instead includes the full path of the znode. Not only does this choice simplifies the API (no open() or close() methods), but it also eliminates extra state that the server would need to maintain.

## Zookeeper Guarantees

Two guarantees: **Linearizable writes and FIFO ordering to client requests.** 
Linearizable writes means that any each update takes place at one point in time and all updates have a global order. All operations appear to be instantaneous/atomic. All updates from client are processed in order, even though there are outstanding requests. All read requests are processed at each replica. They are not linearizable.

- When a new leader is making changes, we don’t want the clients to start using the configuration while its being changed. And if new leader dies, we don’t want clients to use partial config.
- Because of the ordering guarantees, if a process sees the ready znode, it must also see all the configuration changes made by the new leader. If the new leader dies before the ready znode is created, the other processes know that the configuration has not been finalized and do not use it.
- **Notification ordering** is that client will get the notification before it sees the new changes that are made.

> The new leader will update the configuration first, and only then create READY znode. Since all the requests are executed in FIFO order, that means all individual updates are processed before ready znode is created. When the ready appears, linearizable write guarantee will ensure that updates on ALL nodes have happened. If the process dies before updating all the configuration, then ready node will never be created.

Because of the ordering guarantees, if a process sees the ready znode, it must also see all the configuration changes made by the new leader. If the new leader dies before the ready znode is created, the other processes know that the configuration has not been
finalized and do not use it. 

## Examples of Primitives

- If two processes share configuration and also talk to each other through a separate channel, in that case if a change is made by A, then B might not see it, if the B’s ZK replica is behind. In such cases B should issue a sync request. Its a slow read but it will let all the pending writes to be applied.
- ZK is available as long as Quorum is present, even if some nodes fail.
- Ordering guarantees provide base for system’s consistent state, and watches provide base for waiting (getting notification only when the state is changed).
- **Configuration management**: Use a node for storing configuration and processes get config by reading it, and put a watch to be notified if config is updated. After getting the new config, process again puts the watch flag on Zc.

**Rendezvous**: it is when some processes have to consume some information and some other process had to produce but until that information is not their the consumers can not read. We can create a Zr node which is filled by the master as it starts, say with IPs and Ports. When workers start they read that node.

**Group Membership**: We can use ephemeral nodes to implement group membership. Ephemeral nodes are tied to session that created them. Each member of group creates an ephemeral node under group Zg node. If process ends/fails, the znode of that process is removed automatically. Processes can list members by listing the children of Zg. To monitor changes in membership, processes can set watch flag and they will be notified only when info is updated.

**Simple Lock**: Client creates a znode with an ephemeral flag. This method uses so called “lock files”, to implement lock. If a client tries to create and succeed then it holds the lock. Other clients set a watch on the node. It is prone to herd effect when the current process release lock.

**Lock without Herd Effect**: we define a lock-znode and all clients wanting to have a lock line up under this znode in the order of their arrival. All clients have ephemeral and sequential flags set. Use of sequential flag orders the clients attempt to acquire the lock. Only the client with the lowest sequence number holds the lock. Client waits for the deletion of the znode that holds the lock. By only watching for one znode that precedes the client’s znode we avoid the herd effect. There is no polling or timeout.

**Read/Write locks** is implemented by sharing the read lock among many readers. For a writer to get lock, it must wait for the node before it, and wait until it gets deleted. For a read node, it must only wait if there exists a write lock smaller than its znode number, otherwise it can share lock with other read nodes.