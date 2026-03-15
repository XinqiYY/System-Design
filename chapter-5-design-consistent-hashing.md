# Chapter 5: DESIGN CONSISTENT HASHING

## Overview

To achieve horizontal scaling, it is important to distribute requests/data efficiently and evenly\
across servers. Consistent hashing is a commonly used technique to achieve this goal. But\
first, let us take an in-depth look at the problem.

### The rehashing problem

If you have n cache servers, a common way to balance the load is to use the following hash\
method:

> serverIndex = hash(key) % N, where N is the size of the server pool.

This approach works well in even situations, but problems arise when new servers are added or deleted, which will cause the cache client to connect to the wrong servers to fetch data and cause cache misses. Consistent hashing is an effective technique to mitigate this problem.

Quoted from Wikipedia: "Consistent hashing is a special kind of hashing such that when a\
hash table is re-sized and consistent hashing is used, only k/n keys need to be remapped on\
average, where k is the number of keys, and n is the number of slots. In contrast, in most\
traditional hash tables, a change in the number of array slots causes nearly all keys to be\
remapped"

## Hash ring structure and operation

### Hash space and hash ring

Hash space means capacity.  The hash ring is collecting both ends.

<figure><img src=".gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

### Hash servers

Using the same hash function f, we map servers based on server IP or name onto the ring.\
Figure 5-5 shows that 4 servers are mapped on the hash ring.

<figure><img src=".gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

### Hash keys

One thing worth mentioning is that hash function used here is different from the one in “the\
rehashing problem,” and there is no modular operation. As shown in Figure 5-6, 4 cache keys\
(key0, key1, key2, and key3) are hashed onto the hash ring

<figure><img src=".gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

### Server lookup

To determine which server a key is stored on, we go clockwise from the key position on the\
ring until a server is found

<figure><img src=".gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

### Add a server

Adding a new server will only require the redistribution of a fraction of keys.

> After a new server 4 is added, only key0 needs to be redistributed. k1, k2, and\
> k3 remain on the same servers. Let us take a close look at the logic. Before server 4 is added,\
> key0 is stored on server 0. Now, key0 will be stored on server 4 because server 4 is the first\
> server it encounters by going clockwise from key0’s position on the ring. The other keys are\
> not redistributed based on consistent hashing algorithm.

<figure><img src=".gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

### Remove a server

When a server is removed, only a small fraction of keys require redistribution with consistent\
hashing.&#x20;

> When server 1 is removed, only key1 must be remapped to server 2.\
> The rest of the keys are unaffected.

<figure><img src=".gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

## Two issues in the basic approach

The basic steps are:

* Map servers and keys on to the ring using a uniformly distributed hash function.
* To find out which server a key is mapped to, go clockwise from the key position until the\
  first server on the ring is found.

However, two problems with this approach:

1. It is impossible to keep the same size of partitions on the ring for all servers, considering a server can be added or removed.
   1.  > if s1 is removed, s2’s partition (highlighted with the bidirectional arrows) is twice as large as s0 and s3’s partition.

       <figure><img src=".gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>
2. Keys may not be distributed evenly.
   1.  > most of the keys are stored on server 2, server 1 and server 3 have no data.

       <figure><img src=".gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

### Solution

A technique called virtual nodes or replicas is used to solve these problems.

#### Virtual nodes

Each server is represented by multiple virtual nodes and is responsible for multiple partitions on the ring.

> Partitions (edges) with label s0 are managed by server 0. Label s1 are managed by server 1

<figure><img src=".gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

To find which server a key is stored on, we go clockwise from the key’s location and find the first virtual node encountered on the ring

As the number of virtual nodes increases, the distribution of keys becomes more balanced. However, more spaces are needed to store data about virtual nodes. This is a tradeoff, and we can tune the number of virtual nodes to fit our system requirements.

#### Find affected keys

> How can we find the affected range to redistribute the keys?

Image below, server 4 is added onto the ring. The affected range starts from s4 (newly\
added node) and **moves anticlockwise** around the ring until a server is found (s3). Thus, **keys**\
**located between s3 and s4 need to be redistributed to s4**

<figure><img src=".gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

Same as deleting. When a server (s1) is removed, the node from s1 (the removed node) moves anticlockwise, and the keys located between s0 and s1 must be redistributed to s2

<figure><img src=".gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

## Benefits of consistent hashing:

* Minimized keys are redistributed when servers are added or removed.
* It is easy to scale horizontally because the data are more evenly distributed.
* Mitigate the hotspot key problem. Excessive access to a specific shard could cause the server\
  overload. Imagine data for Katy Perry, Justin Bieber, and Lady Gaga all end up on the\
  same shard. Consistent hashing helps to mitigate the problem by distributing the data more evenly

## Used on:

* Partitioning component of Amazon’s Dynamo database
* Data partitioning across the cluster in Apache Cassandra
* Discord chat application
* Akamai content delivery network
* Maglev network load balancer

