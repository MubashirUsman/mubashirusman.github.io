---
layout: post
title:  "SRE Book Sayings"
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


