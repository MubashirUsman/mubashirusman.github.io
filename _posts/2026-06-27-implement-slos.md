---
layout: post
title:  "Book: SRE Workbook"
date:   2026-06-27 00:00:00 +0000
categories: distributed-systems sre
---

# Chapter 2 Implementing SLOs

Finding a balance between investing in new functionality and reliability for the exisiting features is hard. Therefore SLOs are important and day to day work of SREs is driven by SLOs at Google. Error budget should be a decision making tool and not just another KPI. For this product owners should agree upon the SLOs that fit the product and people responsible for meeting the SLO should agree that its possible to achieve it under normal conditions. 

SLOs are the target level of reliability that makes customers unhappy if not met. Reliability target of 100% is not practical for a number of reasons. It can not be achieved even using automated failover, high availability and healthchecking. There are number of components involved between service and the user and any of them can fail. Its impossible to improve the service if reliability is to be maintained at 100%.

An SLI is an indicator of the service level being provided, `SLI= number_of_good_events/total_events`. 

```
Number of successful HTTP requests / total HTTP requests

gRPC requests completed in < 100ms / total gRPC requests

Number of search results that used the entire corpus / total number of search results, including those that degraded gracefully

stock_check_count that used fresher data than 10 mins / total stock checks

Throughput per second grearter than 100GB / total throughput including less than 100GB

```

Thinking SLO in terms of percentage between 0 and 100 is intutive and makes it easy to define error budget. Error budget is 100 minus the target SLO. E.g if you have 3 million requests with an SLO of 99.9% successful rate per 4 weeks window, then there is an error budget of 3000 requests over the 4 week period. Two steps in attempting to define SLIs: __spcecification and implementation__. For example, loading home page that completes in less than 100ms can be measured from the server logs, or from the client side Javascript library. 

Availability, latency, correctness, durability are common SLOs. Choose an SLI which is relevant but also easy to measure.

## Types of Elements and SLI specification

Components can be divided into three types: **Request Driven** are the components where a user creates an event and expects a response. **Pipeline** systems that takes in inputs, do some processing, and write the output somehwere. E.g a process that process logs and generate reports, a process that reads from database and writes to a distributed hash table. **Storage** systems that take in data and make it available for future retrieval: e.g are databases or file systems.

SLIs for a request driven service could be _availability_ (the ratio of successful requests), _latency_ (portion of requests completed in less than X milliseconds) and _Quality_ (how ratio of undegraded responses to sum of degraded and undegraded responses ).

SLIs for a pipeline may be _freshness_ the amount of data updated recently than the agreed time (like scores updated in 10 mins), _coverage_ data processed above a certain target amount by batch process or ratio of amount of records processed within a certain time for stream processing.

SLIs for storage could be _throughput_ and _durability_.

## SLI Implementation

For the first SLIs, take something that requires minimum work. You need enough information to measure the SLI: for availability, you need the success/failure status; for slow requests, you need the time taken to serve the request.

**API and HTTP server availability and latency** needs to be measured. For availability we form the HTTP response codes as a measure of success or failed request. Any 5XX code are counted as failed requests and any other responses are successful. Latency SLI is the ratio of requests faster than a threshold.
Sources for this information could be _Application server logs_, _Load balancer monitoring_, or _Black-box monitoring_. The book uses load balancer monitoring to measure these SLIs.

**Pipeline freshness, coverage and correctness** could be measured. For freshness, increment a metric counter saying that data was requested. Increment another counter if the data was fresher than a predefined threshold. The book uses client side implementation for freshness SLI. For coverage SLI, measure the number of records processed successfully processed vs the number of records that it should have processed. For correctness, either inject correct results and count the proportion of times the output matches expectations or use a different pipeline to measure the correct results and compare them with ours. 

## Measuring the SLIs

```
Availability = sum(rate(http_requests_total{host="api", status!~"5.."}[7d])) / sum(rate(http_requests_total{host="api"}[7d]))

Latency = histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[7d]))

Freshness = count of all data_requests for "api" and "web" with freshness ≤ 1 minute / count of all data_requests

Correctness = count of all data_requests which were correct / count of all data_requests
```
