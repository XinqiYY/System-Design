# Chapter 4: DESIGN A RATE LIMITER

## Overview

In a network system, a rate limiter is used to control the rate of traffic sent by a client or a service. In the HTTP world, a rate limiter limits the number of client requests allowed to be sent over a specified period. If the API request count exceeds the threshold defined by the rate limiter, all the excess calls are blocked.

> Why the rate limiter is needed?

1. Prevent resource starvation caused by Denial of Service(DoS) attacks.
2. Reduce cost.
3. Prevent servers from being overloaded.

> Hard vs soft rate limiting.&#x20;

* Hard: The number of requests cannot exceed the threshold.&#x20;
* Soft: Requests can exceed the threshold for a short period.

## Questions can be asked

* Client-side, server-side, middleware
* inform users who are throttled

## Where to put the rate limiter?

* Client-side: Generally speaking, client is an unreliable place to enforce rate limiting because client requests can easily be forged by malicious actors. Moreover, we might not have control over the client implementation.
*   Server-side:

    <figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>
*   Middleware:&#x20;

    <figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

> API Gateway

Cloud microservices \[4] have become widely popular and rate limiting is usually implemented within a component called API gateway. API gateway is a fully managed service that supports rate limiting, SSL termination, authentication, IP whitelisting, servicing static content, etc.

## Algorithms for rate limiting

### Token bucket

A token bucket is a container that has pre-defined capacity. Tokens are put in the bucket at preset rates periodically. Each request consumes one token. When a request arrives, we check if there are enough tokens in the bucket. If yes, take one token out for each request, else the request is dropped.

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

The token bucket algorithm takes two parameters:

* Bucket size: the maximum number of tokens allowed in the bucket.
* Refill rate: number of tokens put into the bucket every second

| Pros                                      | Cons                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| Easy to implement.                        | It might be challenging to tune the two parameters properly. |
| Memory efficient.                         |                                                              |
| Allows a burst traffic for short periods. |                                                              |

### Leaking bucket

The leaking bucket algorithm is similar to the token bucket except that requests are processed at a fixed rate. It is usually implemented with a first-in-first-out (FIFO) queue. The algorithm works as follows:

* When a request arrives, the system checks if the queue is full. If it is not full, the request is added to the queue.
* Otherwise, the request is dropped.
* Requests are pulled from the queue and processed at regular intervals.

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

Leaking bucket algorithm takes the following two parameters:

* &#x20;Bucket size: it is equal to the queue size. The queue holds the requests to be processed at a fixed rate.
* Outflow rate: it defines how many requests can be processed at a fixed rate, usually in seconds.

| Pros                                                | Cons                                                                                                                                  |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Memory efficient                                    | A burst of traffic fills up the queue with old requests, and if they are not processed in time, recent requests will be rate-limited. |
| Used in a stable outflow rate since the fixed rate. | It might be challenging to tune the two parameters properly.                                                                          |

### Fixed window counter

Fixed window counter algorithm works as follows:

* The algorithm divides the timeline into fix-sized time windows and assign a counter for each window.
* Each request increments the counter by one.
* Once the counter reaches the pre-defined threshold, new requests are dropped until a new time window starts.

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

> The main problem is that a burst of traffic at the edges of time windows could cause more requests than allowed quota to go through.

<figure><img src=".gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

| Pros                                                                               | Cons                                                                                                      |
| ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Memory efficient                                                                   | Spike in traffic at the edges of a window could cause more requests than the allowed quota to go through. |
| Resetting available quota at the end of a unit time window fits certain use cases. |                                                                                                           |

### Sliding window log

The sliding window log algorithm fixes the issue above:

* The algorithm keeps track of request timestamps. Timestamp data is usually kept in cache, such as sorted sets of Redis.
* When a new request comes in, remove all the outdated timestamps. Outdated timestamps are defined as those older than the start of the current time window.
* Add timestamp of the new request to the log.
* If the log size is the same or lower than the allowed count, a request is accepted. Otherwise, it is rejected.

<figure><img src=".gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

In this example, the rate limiter allows 2 requests per minute.

* The log is empty when a new request arrives at 1:00:01. Thus, the request is allowed.
* A new request arrives at 1:00:30, the timestamp 1:00:30 is inserted into the log. After the insertion, the log size is 2, not larger than the allowed count. Thus, the request is allowed.
* A new request arrives at 1:00:50, and the timestamp is inserted into the log. After the insertion, the log size is 3, larger than the allowed size 2. Therefore, this request is rejected even though the timestamp remains in the log.
* A new request arrives at 1:01:40. Requests in the range \[1:00:40,1:01:40) are within the latest time frame, but requests sent before 1:00:40 are outdated. Two outdated timestamps, 1:00:01 and 1:00:30, are removed from the log. After the remove operation, the log size becomes 2; therefore, the request is accepted.

| Pros                                                                           | Cons                                                                                                         |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| Very accurate. In any rolling window, requests will not exceed the rate limit. | Consumes a lot of memory since even if a request is rejected, its timestamp might still be stored in memory. |

### Sliding window counter

A hybrid approach that combines the fixed window counter and sliding window log.

<figure><img src=".gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

Assume the rate limiter allows a maximum of 7 requests per minute, and there are 5 requests in the previous minute and 3 in the current minute. For a new request that arrives at a 30% position in the current minute, the number of requests in the rolling window is calculated using the following formula:

* Requests in current window + requests in the previous window \* overlap percentage of the rolling window and previous window
* Using this formula, we get 3 + 5 \* 0.7% = 6.5 request. Depending on the use case, the number can either be rounded up or down. In our example, it is rounded down to 6.

Since the rate limiter allows a maximum of 7 requests per minute, the current request can go through. However, the limit will be reached after receiving one more request.

| Pros                                                                                                 | Cons                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| It smooths out spikes in traffic since the rate is based on the average rate of the previous window. | It only works for a not-so-strict lookback window because it approximates the actual rate by aggregating counts from fixed time windows. As a result, requests near the boundaries of these windows may not be evenly distributed, leading to potential inaccuracies. |
| Memory efficient.                                                                                    |                                                                                                                                                                                                                                                                       |

## High-level architecture

<figure><img src=".gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

* Rules are stored on the disk. Workers frequently pull rules from the disk and store them in the cache.
* When a client sends a request to the server, the request is sent to the rate limiter middleware first.
* Rate limiter middleware loads rules from the cache. It fetches counters and last request timestamp from Redis cache. Based on the response, the rate limiter decides:
  * If reached, the rate limiter returns 429 too many requests error to the client and the request is either dropped or forwarded to the queue.
  * If not reached, the request is sent to API servers and increments the counter in Redis.

### Components:

#### Redis:

Tracking how many requests are sent from the same user, IP address, etc.

* INCR: It increases the stored counter by 1.
* EXPIRE: It sets a timeout for the counter. If the timeout expires, the counter is automatically deleted.

> How are rate limiting rules created? Where are the rules stored?

Rules are generally written in configuration files and saved on disk.

> How to handle requests that are rate limited?

In case a request is rate limited, APIs return a HTTP response code 429 (too many requests). We can enqueue the rate-limited requests to be processed later.

> How does a client know whether it is being throttled?&#x20;
>
> And how does a client know the number of allowed remaining requests before being throttled?

The rate limiter returns the following HTTP headers to clients:

* X-Ratelimit-Remaining: The remaining number of allowed requests within the window.&#x20;
* X-Ratelimit-Limit: It indicates how many calls the client can make per time window.
* X-Ratelimit-Retry-After: The number of seconds to wait until you can make a request again without being throttled.

## Rate limiter in a distributed environment

There are two challenges to supporting multiple servers and concurrent threads: Race conditions and Synchronization issues.

#### Race conditions:

> Locks are the most obvious solution for solving race conditions. However, locks will significantly slow down the system.&#x20;
>
> Two strategies are commonly used to solve the problem: **Lua script** and **sorted sets data structure in Redis**

#### Synchronization issues:

> Use sticky sessions that allow a client to send traffic to the same rate limiter. This solution is not advisable because it is neither scalable nor flexible.
>
> A better approach is to use centralized data stores like Redis.
>
> ![](<.gitbook/assets/image (16).png>)

## Performance optimization

1. A multi-datacenter that improved latency.&#x20;
2. Synchronize data with an eventual consistency model.

## Monitoring

If the rate limiting rules are too strict, many valid requests are dropped, relax the rules.&#x20;

If the rate limiter is ineffective in burst traffic, replace the algorithm, such as the Token bucket.
