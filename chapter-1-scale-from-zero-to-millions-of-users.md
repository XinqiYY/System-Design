# Chapter 1: SCALE FROM ZERO TO MILLIONS OF USERS

## Steps:

Start with a single server setup

> What if single server is not enought?

Adding more servers, one for web/mobile, one for database.

> Which database to use

See the database section

> What if many users access the web server at the same time and it reaches the load limit?

Use a load balancer.

> What about data tier, we only have one database which is not support failover and redundancy?

Database replication

> How to improve load/response time

Adding a cache layer and shifting static content to the CDN.

> How to scale the web tier horizontally?

Use a stateless server, and the web tier can achieve auto-scaling

> How to support wider geographical areas?

Use multiple data centers

> How to scale independently?

Decouple

> How to monitor the application's health?

By logging and metricx. Also, automate the tool

> What if the database gets more overloaded?

Scale the data tier.

## Single server setup

It's the first step to building a complex system, and everything runs on one server, web app, database, cache, etc.

## Domain Name System DNS

It is usually a paid service provided by 3d parties.

## Relational vs Non-Relational Database

Use NoSQL when:

* Requires super-low latency
* Data is unstructured
* Need to serialize and deserialize data (JSON, XML, YAML, etc.)
* Store a massive amount of data

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F3fTaHqW8wT8jSA4O91y3%2Fuploads%2FuHdAiF5EQzMr2rLqhb3P%2Fimage.png?alt=media&#x26;token=f918b2a3-9afd-493b-93b2-9bb0cf7216b7" alt=""><figcaption></figcaption></figure>

## Load balancer <a href="#cache" id="cache"></a>

A load balancer evenly distributes incoming traffic among web servers that are defined in a load-balanced set. Then the server side is unreachable directly from the client server.

> How do servers communicate with each other?

They use private IPs, which means they must be in the same network.

> What if the current number of servers is not enough?

It only needs to add more servers to the web server pool, and the load balancer automatically starts to send requests to them.

## Database replication: <a href="#cache" id="cache"></a>

### Master/Slave

A master database supports write operations, and a slave database gets copies of the data from the master database and only supports read operations.

Benefits:

* Better performance: read operations are distributed across slave nodes, allowing more queries to be processed in parallel.
* Reliability: data is replicated across multiple locations.
* High availability: The database can be accessed by another database if it is offline.

> What if one of the databases goes offline?

* If only one slave is available, the read operation will be performed from the master database temporarily, and a new slave will be created to replace the old one. If multiple slaves are available, the read operation will go directly to another healthy slave, and a new slave will replace the old one.
* If the master goes offline, a slave will be prompted as the master, and all slaves will be executed on the new master. A new slave will replace the old one for data replication immediately.&#x20;

> What if the prompted slave doesn't contain up-to-date data?

* Run recovery scripts
* Multi-master replication method
* Circular replication method.

## Cache <a href="#cache" id="cache"></a>

A cache is a temporary storage area that stores the result of expensive responses or frequently accessed data in memory so that subsequent requests are served more quickly.

Considerations for using cache:

* Use when read frequently but modified infrequently.
* Expiration policy: Use expiration, don't be too short since the database access increases, and don't be too long since the data can be stale.
* Consistency: Consistency is a challenge when scaling across multiple regions.
* Mitigating failures: To avoid SPOF, either use multiple cache servers across different data centers or overprovision the required memory by certain percentages.
* Eviction Policy: Once the cache is full, any requests to add items to the cache might cause existing items to be removed. This is called cache eviction. Least-recently-used (LRU) is the most popular cache eviction policy. Other eviction policies, such as the Least Frequently Used (LFU) or First in First Out (FIFO), can be adopted to satisfy different use cases.

## CDN:

A CDN is a network of geographically dispersed servers used to deliver static content.

Considerations of using a CDN:

* Cost: It is run by 3rd parties. Do not cache infrequently used assets.
* Expiration: Use the appropriate TTL to avoid repeated loading of content or content that is stale.
* CDN fallback: if CDN fails, request resources from the origin.
* Invalidating file:
  * By APIs
  * Use versioning by adding a parameter to the URL.

## Stateful/Stateless server

A stateful server remembers client data from one request to the next, and a stateless server keeps no state information.

> Issue with stateful server

A stateful server may cause overhead because each client needs to pair with a server, adding or removing servers is also difficult, and it's also hard to handle server failures. Stateless server is the best solution, which fetches state data from a shared data store, which is simpler, robust, and scalable.&#x20;

##

## Data Center

### GeoDNS

GeoDNS is a DNS service that allows domain names to be resolved to IP addresses based on the location of a user.

Challenge in multi-data center setup:

* Traffic redirection: use geoDNS.
* Data synchronization: replicate data across multiple data centers.

## Message queue

A message queue is a durable component, stored in memory, that supports asynchronous communication. It serves as a buffer and distributes asynchronous requests.&#x20;

## Logging, metrics, automation

**Logging**: Monitoring error logs is important because it helps to identify errors and problems in the system.

**Metrics**: Collecting different types of metrics helps us to gain business insights and understand the health status of the system.

* Host-level metrics: CPU, Memory, disk I/O, etc.
* Aggregated level metrics: for example, the performance of the entire database tier, cache tier, etc.
* Key business metrics: daily active users, retention, revenue, etc.

**Automation**: When a system gets big and complex, we need to build or leverage automation tools to improve productivity.&#x20;

## Vertical scaling vs Horizontal scaling <a href="#vertical-scaling-vs-horizontal-scaling" id="vertical-scaling-vs-horizontal-scaling"></a>

### Vertical scaling:

It is referred to as "scale up", which means adding more power to the servers.

**Limitation:**

* It has a hard limit that is unable to add ultimate CPU and memory to a single server.
* It doesn't have failover and redundancy.
* Cost is high

### Horizontal scaling

It is referred to as "scale out," which means scaling by adding more servers to the pool of resources.

Horizontal scaling, also known as sharding, is the practice of adding more servers. Sharding separates large databases into smaller, more easily managed parts called shards. Each shard shares the same schema, though the actual data on each shard is unique to the shard. Anytime you access data, a hash function is used to find the corresponding shard. Sharding key (known as a partition key). Please always choose a key that can evenly distribute data.

### New challenge for sharding: <a href="#new-challenge-for-sharding" id="new-challenge-for-sharding"></a>

1.  Resharding data: Resharding data is needed when&#x20;

    1. a single shard can no longer hold more data due to rapid growth.&#x20;
    2. Certain shards might experience shard exhaustion faster than others due to uneven data distribution.&#x20;

    When shard exhaustion happens, it requires updating the sharding function and moving data around consistent hashing.
2. Celebrity problem: hotspot key problem. Imagine data for famous people; in social applications, we may need to allocate a shard for each celebrity. Each shard might even require further partitioning.
3.  Join and denormalization: Once a database has been sharded across multiple servers, it is hard to perform join operations across database shards. A common workaround is to denormalize the database so that queries can be performed in a single table.

    <figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F3fTaHqW8wT8jSA4O91y3%2Fuploads%2FV9mZDc54cSkkjt7bwSv5%2Fimage.png?alt=media&#x26;token=478654a9-1a43-4320-906f-135eb2e8a4bc" alt=""><figcaption></figcaption></figure>

## Millions of users and beyond

* Keep the web tier stateless
* Build redundancy at every tier&#x20;
* Cache data as much as you can&#x20;
* Support multiple data centers&#x20;
* Host static assets in CDN&#x20;
* Scale your data tier by sharding&#x20;
* Split tiers into individual services&#x20;
* Monitor your system and use automation tools

