---
layout: post
title:  "Paper: The Tail at Scale (2013)"
date:   2026-06-06 00:00:00 +0000
categories: distributed-systems
---

As large scale systems should be made fault-tolerant out of faulty components, similarly large online services should be responsive out of less predictable parts, such services are called **tail-tolerant**.

Tail Latency is the latency caused by slowest queries of the system. It becomes significant if measured at scale and also adds up quickly when request stack cosnsists of many layers. 
There are various causes for tail latency: __shared resources, daemons, globally shared resources such as network switch, garbage collection, SSD garbage collection, power saving mode__.

At scale, even rare hiccups become significant.
> For example if your service has a 99 percentile latency of <200ms, that means 1% of req. take longer than 200ms. But if the request goes through 100 such services to create a response and all services have the 99% latency <200ms, then 64% of the requests will take more than 200ms.

# Reducing latency variability
Overprovisioning of resources, careful real-time engineering of software and improved reliability can help reduce the causes of variability. For example: __prioritize interactive__ requests over batch requests, __reduce head-of-the-line__ blocking by breaking the requests, managing background activities such as triggering heavy operations at the time of lower load. For large fan out services synchronize the background activity across many machines, slowing only requests that are currently being handled instead of continuous tail latency.

# Living with latency
There are techniques to live with latency instead of removing it altogether.
- Within request immediate response (scale of mili seconds)
- Cross request long term (scale of seconds to minutes)

## Within Request Short Term Adaptation
Mostly effective on read-only, loosly consistent datasets such as spelling correction service.

### Hedged Requests
One way to curb latency variability is to issue the same request to multiple replicas and use the results from whichever server replies first. It can implemented by sending a first request to the replica which is believed to be fast, but then sending it with a breif delay. However it can increase the number of requests that will be sent significantly if used naively. A smarter way is to defer sending the secondary request until the first request has been outstanding for longer than 95th percentile latency. This limits additional requests to only 5%. In this way top 5% of the long standing requests will be served faster.
**Retry only the ones taking longer than 95%tile of time**

### Tied Requests
Hedged requests have the vulnerability in which multiple servers can execute the request unnecessarily. Even though it can be reduced by sending only most latent requests but it still serves limited number of requests. Load balancing of requests should be done based on outstanding requests for a given server, instead of random technique. In fact, instead of choosing a server enqueue requests in multiple servers simultaneously with a tag of `tied` request. Then server updates its counterpart when it starts to execute the request.

> Enqueuing requests to multiple servers with a delay of 2*network_message_delay_between_servers, so that both servers do not start executing at the same time. Then if server A starts to process the request, it updates the status on another server.

