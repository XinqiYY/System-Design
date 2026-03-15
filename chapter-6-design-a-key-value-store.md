# Chapter 6: DESIGN A KEY-VALUE STORE

## Overview

Key-value store = non-relational database.

In this chapter, you are asked to design a key-value store that supports the following operations:

* put(key, value) // insert “value” associated with “key”
* get(key) // get “value” associated with “key”

### Requirements:

* The size of a key-value pair is small: less than 10 KB.
* Ability to store big data.
* High availability: The system responds quickly, even during failures.
* High scalability: The system can be scaled to support large data set.
* Automatic scaling: The addition/deletion of servers should be automatic based on traffic.
* Tunable consistency.
* Low latency.

## Different stores

### Single server key-value store

In a single server, data can be stored in memory and a hash table. But limited by space constraints. Two solutions:

1. Data compression
2. Store only frequently used data in memory and the rest on disk

Even with these optimizations, a single server can reach its capacity very quickly. A\
distributed key-value store is required to support big data.

### Distributed key-value store

A distributed key-value store is also called a distributed hash table, which distributes key-value pairs across many servers. When designing a distributed system, it is important to\
understand the CAP (Consistency, Availability, Partition Tolerance) theorem.

#### CAP theorem

CAP theorem states it is impossible for a distributed system to simultaneously provide more\
than two of these three guarantees: consistency, availability, and partition tolerance.

* **Consistency**:
  * Same data at the same time, no matter which node they connect to.
* **Availability**:&#x20;
  * User always gets a response, even if some of the nodes are down.
* **Partition Tolerance**:&#x20;
  * A partition indicates a communication break between two nodes. Partition tolerance means the system continues to operate despite network partitions

<figure><img src=".gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

* CP (consistency and partition tolerance) systems: sacrificing availability.
* AP (availability and partition tolerance) systems: sacrificing consistency.
* CA (consistency and availability) systems: sacrificing partition tolerance.

> Since network failure is unavoidable, a distributed system must tolerate network partition. Thus, a CA system cannot exist in realworld applications.

## A concrete example:

In distributed systems, data is usually replicated multiple times. Assume data are replicated on three replica nodes, n1, n2 and n3.

<figure><img src=".gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

**Idea situation**

Data written to n1 is automatically replicated to n2 and n3. Both consistency and availability are achieved.

**Real-world distributed systems**

In a distributed system, partitions cannot be avoided, and when a partition occurs, we must\
choose between consistency and availability. In image, n3 goes down and cannot\
communicate with n1 and n2. If clients write data to n1 or n2, data cannot be propagated to\
n3. If data is written to n3 but not propagated to n1 and n2 yet, n1 and n2 would have stale\
data.

<figure><img src=".gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

Choosing the right CAP that fits your use case is an important step in building a distributed key-value store. Discuss this with your interviewer and design the system accordingly.

## System components

### Data partition

There are two challenges while partitioning the data across multiple servers.

1. Distribute data across multiple servers evenly.
2. Minimize data movement when nodes are added or removed.

Use consistent hashing. Advantages:

* Auto scaling
* Heterogeneity: the number of virtual nodes for a server is proportional to the server capacity. Servers with higher capacity are assigned more virtual nodes.

### Data replication

To achieve high availability and reliability, data must be replicated asynchronously over N servers. After a key is mapped to a position on the hash ring, walk clockwise from that position and choose the first N servers on the ring to store data copies. In Figure 6-5 (N = 3), key0 is replicated at s1, s2, and s3.

<figure><img src=".gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

> How about virtual nodes?

With virtual nodes, the first N nodes on the ring may be owned by fewer than N physical servers. To avoid this issue, we only choose unique servers while performing the clockwise walk logic.

> What if data center fail due to power outages?

Replicas are placed in distinct data centers, and data centers are connected through high-speed networks

### Consistency

Data is replicated at multiple nodes, synchronized across replicas by Quorum consensus.

N = The number of replicas\
W = A write quorum of size W. For a write operation to be considered as successful, write\
operation must be acknowledged from W replicas.\
R = A read quorum of size R. For a read operation to be considered as successful, read\
operation must wait for responses from at least R replicas.

<figure><img src=".gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

The configuration of W, R and N is a typical tradeoff between latency and consistency.

> How to configure N, W, and R to fit our use cases?

If R = 1 and W = N, the system is optimized for a fast read.\
If W = 1 and R = N, the system is optimized for fast write.\
If W + R > N, strong consistency is guaranteed (Usually N = 3, W = R = 2).\
If W + R <= N, strong consistency is not guaranteed.

Need to tune the values of W, R, and N to achieve the desired level of consistency.

#### Consistency models

A consistency model defines the degree of data consistency, and a wide spectrum of possible consistency models exists:

* Strong consistency: any read operation returns a value corresponding to the result of the\
  most updated write data item. A client never sees out-of-date data.
* Weak consistency: subsequent read operations may not see the most updated value.
* Eventual consistency: this is a specific form of weak consistency. Given enough time, all\
  updates are propagated, and all replicas are consistent.

## Inconsistency resolution: versioning

Replication gives high availability but causes inconsistencies among replicas. Versioning and vector locks are used to solve inconsistency problems.

**Vector clock**

A vector clock is a \[server, version] pair associated with a data item. It can be used to check if one version precedes, succeeds, or is in conflict with others.

Two downsides:

1. Vector clocks add complexity to the client because it needs to implement conflict resolution\
   logic
2. The \[server: version] pairs in the vector clock could grow rapidly. To fix this problem, we set a threshold for the length, and if it exceeds the limit, the oldest pairs are removed. This can lead to inefficiencies in reconciliation because the descendant relationship cannot be determined accurately. However, based on the Dynamo paper \[4], Amazon has not yet encountered this problem in production; therefore, it is probably an acceptable solution for most companies.

### Handling failures

#### Failure detection

In a distributed system, it is insufficient to believe that a server is down because another\
server says so. Usually, it requires at least two independent sources of information to mark a\
server down. Fixed by the gossip protocol, which is a decentralized failure detection method.

**Gossip protocol**

* Each node maintains a node membership list, which contains member IDs and heartbeat counters.
* Each node periodically increments its heartbeat counter.
* Each node periodically sends heartbeats to a set of random nodes, which in turn propagate to another set of nodes.
* Once nodes receive heartbeats, membership list is updated to the latest info.
* If the heartbeat has not increased for more than predefined periods, the member is considered as offline

<figure><img src=".gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

#### Handling temporary failures

**Sloppy quorum**

It is used to improve availability. Instead of enforcing the quorum requirement, the system chooses the first W healthy servers for writes and the first R healthy servers for reads on the hash ring. Offline servers are ignored.

**hinted handoff**

If a server is unavailable due to network or server failures, another server will process requests temporarily. When the down server is up, changes will be pushed back to achieve data consistency. This process is called hinted handoff. Since s2 is unavailable in Figure 6-12, reads and writes will be handled by s3 temporarily. When s2 comes back online, s3 will hand the data back to s2.

<figure><img src=".gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

> Hinted handoff is used to handle temporary failures. What if a replica is permanently\
> unavailable?

Use anti-entropy protocol to keep replicas in sync/

**Anti-entropy**

It compares each piece of data on replicas and updates each replica to the newest version. A **Merkle tree** is used for inconsistency detection and minimizing the amount of data transferred.

**Merkle tree**

A hash tree or Merkle tree is a tree in which every non-leaf node is labeled with the hash of the labels or values (in case of leaves) of its child nodes. Hash trees allow efficient and secure verification of the contents of large data structures.

To compare two Merkle trees, start by comparing the root hashes. If root hashes match, both\
servers have the same data. If root hashes disagree, then the left child hashes are compared,\
followed by right child hashes. You can traverse the tree to find which buckets are not\
synchronized and synchronize those buckets only

#### Handling a data center outage

To build a system capable of handling data center outages, it is important to replicate data\
across multiple data centers. Even if a data center is completely offline, users can still access\
data through the other data centers.

## System architecture diagram

<figure><img src=".gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

* Clients communicate with the key-value store through simple APIs: get(key) and put(key, value).
* A coordinator is a node that acts as a proxy between the client and the key-value store.
* Nodes are distributed on a ring using consistent hashing.
* The system is completely decentralized so adding and moving nodes can be automatic.
* Data is replicated at multiple nodes.
* There is no single point of failure as every node has the same set of responsibilities.

As the design is decentralized, each node performs many tasks as presented in image:

<figure><img src=".gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

## Write path

Based on the architecture of Cassandra. The image explains what happens after a write request is directed to a specific node.

<figure><img src=".gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

1. The write request is persisted on a commit log file.
2. Data is saved in the memory cache.
3. When the memory cache is full or reaches a predefined threshold, data is flushed to SSTable on disk. Note: A sorted-string table (SSTable) is a sorted list of \<key, value> pairs.

## Read path

After a read request is directed to a specific node, it first checks if data is in the memory cache. If so, the data is returned to the client.

<figure><img src=".gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>

If the data is not in memory, it will be retrieved from the disk instead. We need an efficient\
way to find out which SSTable contains the key. **Bloom filter** is commonly used to solve\
this problem

<figure><img src=".gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>

1. The system first checks if the data is in memory. If not, go to step 2.
2. If data is not in memory, the system checks the bloom filter.
3. The bloom filter is used to figure out which SSTables might contain the key.
4. SSTables return the result of the data set.
5. The result of the data set is returned to the client.

## Summary

<figure><img src=".gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>
