# Chapter 1 SCALE FROM ZERO TO MILLIONS OF USERS

## Relational vs Non-Relational Database

![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F3fTaHqW8wT8jSA4O91y3%2Fuploads%2FuHdAiF5EQzMr2rLqhb3P%2Fimage.png?alt=media\&token=f918b2a3-9afd-493b-93b2-9bb0cf7216b7)

## Vertical scaling vs Horizontal scaling <a href="#vertical-scaling-vs-horizontal-scaling" id="vertical-scaling-vs-horizontal-scaling"></a>

Horizontal scaling, also known as sharding, is the practice of adding more servers.Sharding separates large databases into smaller, more easily managed parts called shards. Each shard shares the same schema, though the actual data on each shard is unique to the shard. Anytime you access data, a hash function is used to find the corresponding shard. Sharding key (known as a partition key)

### New challenge for sharding: <a href="#new-challenge-for-sharding" id="new-challenge-for-sharding"></a>

1.  Resharding data: Resharding data is needed when&#x20;

    1. a single shard could no longer hold more data due to rapid growth.&#x20;
    2. Certain shards might experience shard exhaustion faster than others due to uneven data distribution.&#x20;

    When shard exhaustion happens, it requires updating the sharding function and moving data around consistent hashing.
2. Celebrity problem: hotspot key problem. Imagine data for famous people; in social applications, we may need to allocate a shard for each celebrity. Each shard might even require further partition.
3.  Join and de-normalization: Once a database has been sharded across multiple servers, it is hard to perform join operations across database shards. A common workaround is to de-normalize the database so that queries can be performed in a single table.

    <figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F3fTaHqW8wT8jSA4O91y3%2Fuploads%2FV9mZDc54cSkkjt7bwSv5%2Fimage.png?alt=media&#x26;token=478654a9-1a43-4320-906f-135eb2e8a4bc" alt=""><figcaption></figcaption></figure>

## Database replication: <a href="#cache" id="cache"></a>

A master database supports write operations, and a slave database gets copies of the data from the master database and only supports read operations.

Benefits:

* Better performance: read operations are distributed across slave nodes, allowing more queries to be processed in parallel.
* Reliability: data is replicated across multiple locations.
* High availability: The database can be accessed in another database if it is offline.

## Cache <a href="#cache" id="cache"></a>

Considerations for using cache:

* Use when read frequently but modified infrequently.
* Expiration policy: Use expiration, don't be too short since the database access increases, and don't be too long since the data can be stale.
* Consistency: Consistency is a challenge when scaling across multiple regions.
* Mitigating failures: To avoid SPOF, either use multiple cache servers across different data centers or overprovision the required memory by certain percentages.
* Eviction Policy: Once the cache is full, any requests to add items to the cache might cause existing items to be removed. This is called cache eviction. Least-recently-used (LRU) is the most popular cache eviction policy. Other eviction policies, such as the Least Frequently Used (LFU) or First in First Out (FIFO), can be adopted to satisfy different use cases.

## CDN:

Considerations of using a CDN:

* Cost: do not cache infrequently used assets.
* Expiration: Use the appropriate TTL to avoid repeat loading of content or content that is stale.
* CDN fallback: if CDN is failed, request resources from the origin.

## Data Center

### GeoDNS

GeoDNS is a DNS service that allows domain names to be resolved to IP addresses based on the location of a user.

Challenge in muti-data center setup:

* Traffic redirection: use geoDNS.
* Data synchronization: replicate data across multiple data centers.

## Message queue

A message queue is a durable component, stored in memory, that supports asunchronous communication. It serves as a buffer and distributes asynchronous requests.&#x20;

## Logging, metrics, automation

**Logging**: Monitoring error logs is important because it helps to identify errors and problems in the system.

**Metrics**: Collecting different types of metrics help us to gain business insights and understand the health status of the system.

* Host level metrics: CPU, Memory, disk I/O, etc.
* Aggregated level metrics: for example, the performance of the entire database tier, cache tier, etc.
* Key business metrics: daily active users, retention, revenue, etc.

**Automation**: When a system gets big and complex, we need to build or leverage automation tools to improve productivity.&#x20;

## Millions of users and beyond

* Keep web tier stateless
* Build redundancy at every tier&#x20;
* Cache data as much as you can&#x20;
* Support multiple data centers&#x20;
* Host static assets in CDN&#x20;
* Scale your data tier by sharding&#x20;
* Split tiers into individual services&#x20;
* Monitor your system and use automation tools

