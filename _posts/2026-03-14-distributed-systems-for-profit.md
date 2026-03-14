---
---
# Notes from Distributed Systems for Fun and Profit

## Chapter 1: Basics

Distributed systems is a way of doing a task with many computers instead of doing using one. To do this we have a constraint, that is to do using comodity hardware isntead of relying on most expensive hardware. This is because of a fundamental reason that as the nodes grow the performance difference between high-end hardware and comodity hardware decreases. So everything happens when scale grows, and the goal is to gain the scalability. Scalability can be defined as the ability to grow the system with the amount of work. This can be in terms of data size, computers to administrators ratio, decrease in latency with growing nodes etc.  

Scaleable systems have two properties: 1. Performance (latency) 2. Availability (fault-tolerance)

- Performance is achieving shorter response time OR high throughput OR low utilization of resources.   
In these three latency is interesting as it has little to do with financial limitations and more with physical limitations. Speed of light and hardware components at which they can work is a hard limit. E.g the time between the write initiated and confirmation response received.  
- Availability is proportion of the system functioning properly. If a user can not load the page the system is not available.  
Availability can be measured in terms in uptime, like a 90% availability means more than a month downtime per year. And 99.9% < 9 hours, 99.99% < less than an hour. Availability can be affected by many other factors than just the uptime of a service, like hard disks catching fire, a star falling from a sky or company going bankrupt. The best we can do is to design for fault tolerance. Fault tolerance = the ability to behave well when a fault occurs.

The hinderance between the above good things can be: increased number of nodes increases probability of failure of one node (fault-tolerance), increased number of nodes may result in more communcation (thus reduced performance, latency). Our system design options are beyond these physical constraints. Both performance and availability are defined by external guarantees such as SLAs. 

Making appropriate __abstractions__ for complex systems make them more manageable and understandable. In this regard __models__ can help concretely define what the properties of our system will be. Good abstractions remove the irrelevant details. Some models are: failure modes (crash/byzantine), system model (synchronous/asynchronous), consistency (weak/strong).
A system with weaker guarantees can be more performant/be available and also hard to reason about at the same time.  
Some failures types such as network latencies and network partitions means that a system has to make hard choices between whether its worth it to stay available and provide lose guarantees or reject the requests and play safe.

Design techniques: Partition and replicate  
1. Partition - means to divide data on multiple nodes and each partition is a subset of data. This help to improve performance by limiting the amount of data to work with in a partition. And increase availability by allowing partitions to fail independently.
2. Replication: Copying or reproducing something is something that we can use to fight latency. It improves performance by making the additional bandwidth and compute available. And it improves availability by increasing the copies of data by increasing the number of nodes which should fail before system become unavailable.  
Replicate to reduce the risk of single point of failure. Replicate data to a local cache to reduce latency or on multiple machines to increase throughput. Downsides of replication is that this data needs to be in sync, this means we need to make sure replication follow some consistency model.
Stronger consistency allows you to program the system as if the underlying system was not replicated. But other weaker consistency models expose some underlying details of the system and can offer lower latency and high availability.


## Chapter 2: Up and Down the Level of Abstraction

The fundamental tension is between how we want the system to behave, i.e as a single unit and how the system actually is, distributed. So we create abstractions, we assume that two nodes are equal even when they are not, this makes things easier and manageable. __Impossible results__ tell us that within our assumptions some things are impossible. In a distributed system, programs run concurrently on independent nodes, there is an unreliable network between them, and they have no shared memory or shared clock. This means that the knowledge in a particular node is local, any information about global state is normally out of date, clocks are not synchronized, nodes can fail and recover from a failure independently. A robust system would be that makes little or no assumtions. And we can also make a system with strong assumptions, e.g nodes do not fail a big assumption and system will not need to handle node failure, though this unrealistic assumption.  
- Nodes can fail by __crashing__ or in any arbitrary way other than crashing (__Byzantine fashion__). We only consider crash failure because we can't account infinite number of failures and then design our algorithm.  
- Communication links can be assumed to be __unreliable__ and subject to __message losts__/delays. A __network partition__ occurs when a network fails between nodes but nodes continue to be operational. These are enough assumptions without going into details of individual network links or counting distance between nodes (in a local network).  
- Timing assumptions are essential as the nodes have their own clocks and are at some distance from each other. Synchronous system where there exist __upper bound on message transmission delays__, and asynchronous where processes execute independently without any upper bound. Synchronous means that two processes have the same experience and messages sent will be received within a maximum delay, and processes execute in a lock step. Asynchronous assumes that we can't rely on timing, and assumtions about execution speeds, maximum message delays can help rule out failure scenarios as if they never happened. Real world systems can run occasionally processes within upper bounds but there are certainly times when there are delays and message loss.

### The consensus problem
Some computers are in consensus if they agree on some value. Stages of consensus 1. Agreement: every correct process must agree on a value 2. Integrity: every correct process decides on one value 3. Termination: all processes eventually reach a decision 4. Validity: if all processes reach on a certain value then they all decide that value. Consensus problem helps to solve more advanced problems such atomic commit and atomic broadcast.  
#### FLP Impossibility result
Assumes asynchronous model, it states that we can __not__ have a consensus in an asynchronous system where a process can fail by crashing, even if the messages are never lost. This means that there can not be a consensus if a process remain undecided for an arbitrary amount of time (by delaying message delivery).
#### The CAP theorem
Consistency in CAP means that all computers have the same copy of data or else the system refuses to answer. Availability means that system keeps giving answers even in the face of node failures. Partition tolerance means that system continues to operate even in the face of network division. Only two can be satisfied simultaneusly.
Picking CA: strict quorum protocols such as two phase commit. It can't tolerate any node failure. Strong conistency, it can't differentiate between network partition and node failure. common in traditional relational databases using two phase commit.
Picking CP: includes majority quorum protocols where minority partition is unavailable like Paxos. It can tolerate `n` node failures from `2n+1` nodes, means as long as `n+1` stay up. It makes the minority partition to __NOT__ accept writes and only majority partition can accept.
Picking AP: protocols that involves conflict resolution like DynamoDB
In the face of partition, CAP theorem reduces to __choose between Consistency and Availability__.
Four conclusions from CAP theorem:
- early system designs did not incorporate network partition in their design (mostly CA) but in today's times we can not ignore partitions as systems are spreaded in different geographic regions.
- there is a fundamental tension between strong consistency and high availability when network partition occur.
Strong consistency guarantee require us to give up availability during network partition. Because if we do not give up availability and two nodes can't communicate then we have a divergence. We can overcome this in two ways: 1. Not have partitions 2. weaken the guarantees
- there is a tension between strong consistency and performance in normal operation. Because all nodes must agree on the same result before moving on, this introduces latency. If we can __relax guarantees__ then we can have __less latency__ and __more availability__. If less nodes are involved in an operation we will have less time to wait for the result. Tradeoff here is that we allow to have some anomolies to occur, this means that you read some old data.
Consistency and availability are not binary choices, unless we fix ourselves to __strong consistency__. CAP consistency != ACID consistency.
Consistency is a broader term and strong consistency is just one form of it.
### Consistency models
Can be divided in two categories: 1. Strong consistency models 2. Weak consistency models
- Strong consistency models are: 1. _linearizable consistency_ is the one in which all operations appear to be executed atomically in the same order as the actual time ordering of operations, 2. _sequential consistency_ is same as linearizable except that operations may be executed in a different order than received.
- Weak consistency models are: 1. Client-centric models involve the notion of a client or session in some way. For example forwarding a client to the same replica after they update something so that they don't see older data themselves. 2. Eventual consistency, where all nodes will agree on the same value after an undefined amount of time. Eventually is very weak form of consistency. So lower bound on evntual should be defined. And also how long is eventual.