---
---
## Notes from Distributed Systems for Fun and Profit

### Chapter 1: Basics

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

