# Chapter 2: BACK-OF-THE-ENVELOPE ESTIMATION

Estimate system capacity or performance requirements.

## Power of two:

A byte is a sequence of 8 bits. An ASCII character uses one byte of memory (8 bits)

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

## Latency:

Latency of different computer operations:

<figure><img src=".gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

> ns = nanosecond, µs = microsecond, ms = millisecond\
> 1 ns = 10^-9 seconds\
> 1 µs= 10^-6 seconds = 1,000 ns\
> 1 ms = 10^-3 seconds = 1,000 µs = 1,000,000 ns

**Visualization version:**

<figure><img src=".gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

**Conclusion:**

* Memory is fast, but the disk is slow.
* Avoid disk seeks if possible.
* Simple compression algorithms are fast.
* Compress data before sending it over the internet if possible.
* Data centers are usually in different regions, and it takes time to send data between them.

## Availability numbers:

High availability is the ability of a system to be continuously operational for a desirably long period of time. High availability is measured as a percentage, with 100% means a service that has 0 downtime.

A **service level agreement (SLA)** is a commonly used term for service providers, which is between you (the service provider) and your customer, and this agreement formally defines the level of uptime your service will deliver.

> More nine more better. Most services fall between 99% and 100%

<div data-full-width="true"><figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure></div>

## Tips:

* Use rounding and approximation.
* Write down your assumptions to be referenced later.
* Label your units to avoid confusing yourself.
* Commonly asked back-of-the-envelope estimations: QPS, peak QPS, storage, cache, number of servers, etc.
