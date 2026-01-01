---
layout: post
title:  "What it takes to have minimum reliability?"
date:   2025-12-25 00:30:20 +0100
categories: sre
---
**This architecture is for web-applications and for my own reference.**
Some sane choices to make:
1. Have **public and private subnets** for the infrastructure, This means that application server, database, container registry, object storage, 
logging, monitoring should be inside the *private network*. A firewall should block any 
access from outside to this subnet. In AWS, there is a concept of security groups that can work here.
A load balancer should be in the public subnet.
2. **Database** should be backed up regularly, and for this its important to not let the
single instance of the database to be overloaded. Instead there should be a secondary instance
to take backups from. These backups should be tested by restoring.
3. **Logging** is important for two reasons: to debug after an incident, to keep track of service events and improving it.
A centralized logging solution like ELK stack should be setup. Applications should be configured to collect their logs.
4. A **continous integration and delivery** pipeline is the backbone for quickly testing, releasing in production and rollbacks.
For one service, separate branches should be configured to keep the production code separate from test environments.
Once code is tested, it should be run in a *before-production* environment, make sure that ONLY ONE change is here, and until this
change is released, before-production environment should remain occupied, this will ensure changes to be tested 
and keep the history clean. If the deployment here is unseccessful, its time to go back to testing.
5. **Infrastructure automation** is critical. In a cloud environment, Terraform is my favorite and also an industry standard.
It lets you define your *infrastructure as code*. Also Terraform is declarative in nature, which means it lets you
define what you want at the end and takes into the account the current state, instead of how you should go about to achieve that (imperative definition). Ideally applications should be able to run on stateless servers which effectively 
means that we can deploy identical servers and as many of them as we want. This is the benefit of immutable deployments/containers.
With respect to Terraform, one should templatize the code as variables and modules to reuse for multiple applications and setup a remote state. Lastly, remember manually deploying infrastructure does not scale for a lot of reasons.
6. **Configuration**