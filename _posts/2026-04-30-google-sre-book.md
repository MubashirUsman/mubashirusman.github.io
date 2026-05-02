---
layout: post
title:  "SRE Book"
date:   2026-04-30 00:00:00 +0000
categories: site reliability engineering
---

These are small snippets taken from Site Reliability Engineering book, just for quick reference.

## Chapter 3 Embracing Risk

Users experience is dominated by less reliable components like local internet provider. Cost of making a system reliable is two folds: compute costs and engineers who will focus on reliabilty instead of features. Google strives to make a service reliable enough, but not more than than it needs to be.

#### Measuring Risk
Acceptable level of downtime is the risk a service can tolerate. Usually in terms of uptime measured as nines, `99.9% or 99.99%`.

```
availability = uptime/(uptime+downtime)
```
More appropriate than uptime is **request success rate**, can also be used for systems that are not user facing such as storage system.

```
availability = successful_requests/total_requests
```

Though all failing requests are not equal, such as a new user sign-up failed and a user trying to poll for new email failed are different.

#### Risk Tolerance of a service

A product team is the best resource to discuss the reliability requirements.
- What level of service users expect?
- Is it targetted for consumers or enterprises
- Is it a paid service or free?

E.g Google Apps vs Youtube, have different reliability requirements.

Failures can include planned outages, for example if a service is used in daytimes.

#### Cost
If we have to operate the system at one more 9 of availabilty, will the revenue generated exceed the cost to add one nine, more reliable? Other metrics are also useful, such as latency that can be afforded for a service. Adsense vs Adwords services.

#### Risk Tolerance for Infrastructure services

Infrastructure services have multiple clients with varying needs for same service. One approach is to make all infrastructure services ultra-reliable.

Cost can vary for different needs of clients, one approach is to partition the service and offer it at multiple independent levels of service. For example: BigTable can provisioned with substantial amount of slack capacity to be redundant. Or for high throughput can be provisioned to run very hot and less redundancy.

#### Error budgets
Developers are incentivised to push new code, and SRE are evaluated based upon the reliability offered by a service. Some causes of tension are push frequency, software fault tolerance, testing and canary duration.

Error budget is how much our service is allowed to be unreliable. 

> For this Product owners can define SLO of how much uptime is expected of the service. Then measure uptime, and as long as the uptime measured is above the SLO, there is still error budget remaining. And developers are allowed to release. Developers can strategise how they can use the error budget wisely themselves.

---

## Chapter 4 Service Level Objectives

It takes experience, intution and understanding of what users expect to correctly manage a service. SLI, SLO, SLA define metrics that matter and how to react if we can't meet them.

- SLI is a quantitative measure of some aspect of the level of service being offered. E.g _request latency_: how long it takes to serve a request, _throughput_ how many requests can be served, _errors rate_ over a window of time. _Availability_ is another important SLI measured in number of nines, Google Compute Engine's availability is three and half nines.

- SLO is a target value or range of values for a service level that is measured by an SLI. E.g you want average latency per request to be less than 100ms. Choosing SLOs sets user's expectations about how the service will perform.

- SLA is a contract with your users that includes consequences for not meeting the SLOs. It must have consequences for it to be an SLA. Its normally recognised by paying a financial penality.

#### SLIs in practice
**User facing services** such as Shakespeare care about latency, availability and throughput.  
**Storage systems** such as a filesystems care about latency, availability and durabability. Durability means if the data is still there when we need it.  
**Big data systems** such as data processing pipelines care about throughput and end-to-end latency.

These indicators are collected by monitoring system such as Prometheus, Borgmon typically on server side, sometimes collecting metrics from client side may be interesting.

#### Aggregation
We often aggregate raw measurements, but it can easily hide much higher instantaneous request rates. For example consider two systems with average 100 requests/s, one receiving constant rate while other only gets 200 requests at even seconds and 0 requests on odd seconds. The second system receives twice instantaneous load. Similarly average request latency could appear fast when in reality it contains long tail of requests to be much, much slower. 
Therefore most distributions are **better thought in distributions** rather than average. Using percentiles for indicators allows to consider the shape of distribution, e.g 99th percentile show a possible worst-case value.

We should define the standards for on common definitions of indicators for SLIs. Such as:  

- Aggregation intervals - averaged over 1 minute
- How often measurements are made - every 10 seconds
- Which reuqests are considered - Only GET requests from blackbox monitoring jobs
- How the data is acquired - through our monitoring system measured at the serverd

#### Defining Objectives

Don't start with what what you can measure, instead start working from what users desires are and work backwards to specific indicators. For maximum clarity, SLO's should specify how they are measured and the conditions under which tney are valid.

For example:
```
99% of Get RPC calls will complete in less than 100 ms.

Or specify multiple targets

90% Get RPC calls will complete in less than 1 ms
99% Get RPC calls will complete in less than 10 ms
99.9% Get RPC calls will complete in less than 100 ms
```
> An error budget is just an SLO for meeting other SLOs.!!

While choosing target don't pick based on current performance as you might end up to support a system that needs heroic efforts to sustain. Also keep SLO targets simple, it can be obscure and harder to reason about. Avoid absolute values, such as `always available` is unrealistic, and may need a re-design and be expensive to operate. Keep as few SLOs as possible. **Perfection can wait**, you can always revise the the SLOs over time as you learn more about the system. Start with a loose target and then you can tighten it.

#### SLOs set expectations

Using a tighter internal SLO than SLO advertised to users gives you room to spot chronic problems before they occur. And don't overachieve as this might lead to users over relying on the service. Throttle some requests if system is too fast. Knowing how system is meeting the expectations helps decide whether to invest in making system faster, avaialable or more resilient.