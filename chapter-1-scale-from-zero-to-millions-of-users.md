# Chapter 1 SCALE FROM ZERO TO MILLIONS OF USERS

## Relational vs Non-Relational Database

![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F3fTaHqW8wT8jSA4O91y3%2Fuploads%2FuHdAiF5EQzMr2rLqhb3P%2Fimage.png?alt=media\&token=f918b2a3-9afd-493b-93b2-9bb0cf7216b7)

## Vertical scaling vs Horizontal scaling <a href="#vertical-scaling-vs-horizontal-scaling" id="vertical-scaling-vs-horizontal-scaling"></a>

Horizontal scaling, also known as sharding, is the practice of adding more servers.Sharding separates large databases into smaller, more easily managed parts called shards. Each shard shares the same schema, though the actual data on each shard is unique to the shard. Anytime you access data, a hash function is used to find the corresponding shard. Sharding key (known as a partition key)

### New challenge for sharding: <a href="#new-challenge-for-sharding" id="new-challenge-for-sharding"></a>

Resharding data: Resharding data is needed when 1) a single shard could no longer hold more data due to rapid growth. 2) Certain shards might experience shard exhaustion faster than others due to uneven data distribution. When shard exhaustion happens, it requires updating the sharding function and moving data around, consistent hashing.Celebrity problem: hotspot key problem. Imagine data for famous people; in social applications, we may need to allocate a shard for each celebrity. Each shard might even require further partition.Join and de-normalization: Once a database has been sharded across multiple servers, it is hard to perform join operations across database shards. A common workaround is to de- normalize the database so that queries can be performed in a single table.![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F3fTaHqW8wT8jSA4O91y3%2Fuploads%2FV9mZDc54cSkkjt7bwSv5%2Fimage.png?alt=media\&token=478654a9-1a43-4320-906f-135eb2e8a4bc)

## Cache <a href="#cache" id="cache"></a>

* Use when read frequently but modified infrequently.
* Expiration policy: Use expiration, don't be too short since the database access increases, and don't be too long since the data can be stale.
* Consistency: Consistency is a challenge when scaling across multiple regions.
* Mitigating failures: To avoid SPOF, either use multiple cache servers across different data centers or overprovision the required memory by certain percentages.
* Eviction Policy: Once the cache is full, any requests to add items to the cache might cause existing items to be removed. This is called cache eviction. Least-recently-used (LRU) is the most popular cache eviction policy. Other eviction policies, such as the Least Frequently Used (LFU) or First in First Out (FIFO), can be adopted to satisfy different use cases.
