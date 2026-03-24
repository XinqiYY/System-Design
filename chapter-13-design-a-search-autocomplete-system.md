# Chapter 13: DESIGN A SEARCH AUTOCOMPLETE SYSTEM

## Overview

Autocomplete system, also called “design top k” or “design top k most searched queries”.

## Step 1 - Understand the problem and establish design scope

### Functional requirement

* Matching only support at the begining of a search query
* 5 autocomplete suggestions each time
* Suggestuin return determined by popularity, decided by the historical query frequency
* No, spell check or autocorrect is not supported
* No, we assume all search queries have lowercase alphabetic characters.
* 10 million DAU

### Non-functional requirement

* Fast response time -> within 100 milliseconds
* Relevant
* Sorted -> sorted by popularity or other ranking models
* Scalable
* Highly available

### Back of the envelope estimation

* Assume 10 million daily active users (DAU).
* An average person performs 10 searches per day.
* 20 bytes of data per query string:
  * Assume we use ASCII character encoding. 1 character = 1 byte&#x20;
  * Assume a query contains 4 words, and each word contains 5 characters on average.
  * That is 4 x 5 = 20 bytes per query.
* For every character entered into the search box, a client sends a request to the backend for autocomplete suggestions. On average, 20 requests are sent for each search query. For example, the following 6 requests are sent to the backend by the time you finish typing “dinner”.&#x20;
  * search q=d&#x20;
  * search?q=di
  * search?q=din
  * search?q=dinn
  * search?q=dinne
  * search?q=dinner
* \~24,000 query per second (QPS) = 10,000,000 users \* 10 queries / day \* 20 characters / 24 hours / 3600 seconds.
* Peak QPS = QPS \* 2 = \~48,000
* Assume 20% of the daily queries are new. 10 million \* 10 queries / day \* 20 byte per query \* 20% = 0.4 GB. This means 0.4GB of new data is added to storage daily

## Step 2 - Propose high-level design and get buy-in

At the high-level, the system is broken down into two:

* Data gathering service: It gathers user input queries and aggregates them in real-time. Real-time processing is not practical for large data sets, but we will explore it in a deep dive.
* Query service: Given a search query or prefix, return 5 most frequently searched terms.

### Data gathering service

Assume we have a frequency table that stores the query string and its frequency as shown in image. In the beginning, the frequency table is empty. Later, users enter queries “twitch”, “twitter”, “twitter,” and “twillo” sequentially. Figure 13-2 shows how the frequency table is updated.

<figure><img src=".gitbook/assets/image (92).png" alt=""><figcaption></figcaption></figure>

### Query service

Frequency table, two fields

> * Query: it stores the query string.
> * Frequency: it represents the number of times a query has been searched.

When a user types “tw” in the search box, the following top 5 searched queries are displayed

<figure><img src=".gitbook/assets/image (93).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (94).png" alt=""><figcaption></figcaption></figure>

> But this only suitable for small data set. We will explore optimization in deep dive

## Step 3 - Design deep dive

### Trie data structure

> Why not relational database?

It's inefficient. So use a data structure tree, a prefix tree.

Trie (pronounced “try”) is a tree-like data structure that can compactly store strings. The main idea of trie consists of the following:

* A trie is a tree-like data structure.
* The root represents an empty string.
* Each node stores a character and has 26 children, one for each possible character. To save space, we do not draw empty links.
* Each tree node represents a single word or a prefix string

Basic trie data structure stores characters in nodes. To support sorting by frequency, frequency info needs to be included in nodes.

<figure><img src=".gitbook/assets/image (96).png" alt=""><figcaption></figcaption></figure>

> How does autocomplete work with trie?

* p: length of a prefix
* n: total number of nodes in a trie
* c: number of children of a given node

Steps to get top k most searched queries are listed below:

1. Find the prefix. Time complexity: O(p).
2. Traverse the subtree from the prefix node to get all valid children. A child is valid if it can form a valid query string. Time complexity: O(c)
3. Sort the children and get top k. Time complexity: O(clogc)

The time complexity of this algorithm is the sum of time spent on each step mentioned above:&#x20;

O(p) + O(c) + O(clogc)

> But it need to traverse entire tree, how to optimize?

1. Limit the max length of a prefix
   1. Users rarely type a long search query into the search box. Thus, it is safe to say p is a small integer number, say 50. If we limit the length of a prefix, the time complexity for “Find the prefix” can be reduced from O(p) to O(small constant), aka O(1).
2. Cache top search queries at each node
   1. We store top k most frequently used queries at each node. But it requires a lot of space. Trading space for time is well worth it, as fast response time is very important.
   2. Return top k. Since top k queries are cached, the time complexity for this step is O(1). As the time complexity for each of the steps is reduced to O(1), our algorithm takes only O(1) to fetch top k queries

<figure><img src=".gitbook/assets/image (97).png" alt=""><figcaption></figcaption></figure>

### Data gathering service

Data updated in real-time is not practical because:

* Users may enter billions of queries per day. Updating the trie on every query significantly slows down the query service.
* Top suggestions may not change much once the trie is built. Thus, it is unnecessary to update the trie frequently.

To design a scalable data gathering service, we examine where data comes from and how data is used. Despite the differences in use cases, the underlying foundation for data gathering service remains the same because data used to build the trie is usually from analytics or logging services.

<figure><img src=".gitbook/assets/image (98).png" alt=""><figcaption></figcaption></figure>

#### Analytics Logs

It stores raw data about search queries. Logs are append-only and are not indexed.

<figure><img src=".gitbook/assets/image (99).png" alt=""><figcaption></figcaption></figure>

#### Aggregators

The size of analytics logs is usually very large, and data is not in the right format. We need to aggregate data so it can be easily processed by our system.

> How to aggregate?

Depending on the use case, we may aggregate data differently. For real-time applications such as Twitter, we aggregate data in a shorter time interval as real-time results are important.&#x20;

On the other hand, aggregating data less frequently, say once per week, might be good enough for many use cases. During an interview session, verify whether real-time results are important. We assume trie is rebuilt weekly.

#### Aggregated Data

Assume aggregated weekly data. The “time” field represents the start time of a week. The “frequency” field is the sum of the occurrences for the corresponding query in that week.

<figure><img src=".gitbook/assets/image (100).png" alt=""><figcaption></figcaption></figure>

#### Workers

Workers are a set of servers that perform asynchronous jobs at regular intervals. They build the trie data structure and store it in Trie DB.&#x20;

#### Trie Cache&#x20;

Trie Cache is a distributed cache system that keeps trie in memory for fast read. It takes a weekly snapshot of the DB.&#x20;

#### Trie DB

Trie DB is the persistent storage. Two options are available to store the data:

1. Document store: Since a new trie is built weekly, we can periodically take a snapshot of it, serialize it, and store the serialized data in the database. Document stores like MongoDB \[4] are good fits for serialized data.
2. Key-value store: A trie can be represented in a hash table form \[4] by applying the following logic:
   1. Every prefix in the trie is mapped to a key in a hash table.
   2. Data on each trie node is mapped to a value in a hash table.

<figure><img src=".gitbook/assets/image (101).png" alt=""><figcaption></figcaption></figure>

### Query service

Query service calls the database directly to fetch the top 5 results![](<.gitbook/assets/image (102).png>)

1. A search query is sent to the load balancer.
2. The load balancer routes the request to API servers.
3. API servers get trie data from Trie Cache and construct autocomplete suggestions for the client.
4. In case the data is not in Trie Cache, we replenish data back to the cache. This way, all subsequent requests for the same prefix are returned from the cache. A cache miss can happen when a cache server is out of memory or offline.

Query service requires lightning-fast speed. We propose the following optimizations:

* AJAX request. Usually used in web application which send AJAX requests to fetch autocomplete results. The main benefit of AJAX is that sending/receiving a request/response does not refresh the whole web page.
* Browser caching. Autocomplete suggestions can be saved in the browser cache because data may not change much in a short time. Google search engine uses it.
* Data sampling: For a large-scale system, logging every search query requires a lot of processing power and storage. Data sampling is important. For instance, only 1 out of every N requests is logged by the system

## Trie operations

Trie is a core component of the autocomplete system.&#x20;

**Create**

A trie is created by workers using aggregated data. The source of data is from Analytics Log/DB.

**Update**

There are two ways to update the trie.

1. Update the trie weekly. Once a new trie is created, the new trie replaces the old one.
2. Update the individual trie node directly. We try to avoid this operation because it is slow. However, if the size of the trie is small, it is an acceptable solution. When we update a trie node, its ancestors all the way up to the root must be updated because ancestors store the top queries of children.&#x20;

<figure><img src=".gitbook/assets/image (104).png" alt=""><figcaption></figcaption></figure>

**Delete**

We have to remove hateful, violent, sexually explicit, or dangerous autocomplete suggestions. We add a filter layer in front of the Trie Cache to filter out unwanted suggestions.&#x20;

> Filter layer gives us the flexibility of removing results based on different filter rules. Unwanted suggestions are removed physically from the database asynchronically so the correct data set will be used to build trie in the next update cycle.

<figure><img src=".gitbook/assets/image (105).png" alt=""><figcaption></figcaption></figure>

### Scale the storage

> How to scale it if trie grows too fast and large?

Since English is the only supported language, a naive way to shard is based on the first character.

* If we need two servers for storage, we can store queries starting with ‘a’ to ‘m’ on the first server, and ‘n’ to ‘z’ on the second server.
* If we need three servers, we can split queries into ‘a’ to ‘i’, ‘j’ to ‘r’ and ‘s’ to ‘z’.

> But it creates uneven distribution because letter 'c' more than 'x', how to fix it?

The shard map manager maintains a lookup database for identifying where rows should be stored.

For example, if there are a similar number of historical queries for ‘s’ and for ‘u’, ‘v’, ‘w’, ‘x’, ‘y’ and ‘z’ combined, we can maintain two shards: one for ‘s’ and one for ‘u’ to ‘z’.

<figure><img src=".gitbook/assets/image (106).png" alt=""><figcaption></figcaption></figure>

## Step 4 - Wrap up

Follow up questions:

> Interviewer: How do you extend your design to support multiple languages?

Store Unicode characters in trie nodes.&#x20;

Unicode is an encoding standard that covers all the characters for all the writing systems of the world, modern and ancient \[5].

> Interviewer: What if top search queries in one country are different from others?

We might build different tries for different countries. To improve the response time, we can store tries in CDNs.

> Interviewer: How can we support the trending (real-time) search queries?

Assuming a news event breaks out, a search query suddenly becomes popular. Our original design will not work because:

* Offline workers are not scheduled to update the trie yet because this is scheduled to run on weekly basis.
* Even if it is scheduled, it takes too long to build the trie.

Building a real-time search autocomplete is complicated and is beyond the scope of this book\
so we will only give a few ideas:

* Reduce the working data set by sharding.
* Change the ranking model and assign more weight to recent search queries.
* Data may come as streams, so we do not have access to all the data at once. Streaming data means data is generated continuously. Stream processing requires a different set of systems: Apache Hadoop MapReduce \[6], Apache Spark Streaming \[7], Apache Storm \[8], Apache Kafka \[9], etc.
