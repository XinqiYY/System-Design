# Chapter 9:  DESIGN A WEB CRAWLER

## Overview:

A web crawler is known as a robot or spider. It is widely used by search engines to discover\
new or updated content on the web. Content can be a web page, an image, a video, a PDF\
file, etc. A web crawler starts by collecting a few web pages and then follows links on those\
pages to collect new content

<figure><img src=".gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

A crawler is used for:

* Search engine indexing: A crawler collects web pages to create a local index for search engines. For example, Googlebot is the web crawler behind the Google search engine.
* Web archiving: Collecting information from the web to preserve data for future uses. For instance, many national libraries run crawlers to archive websites. Notable examples are the US Library of Congress and the EU web archive.
* Web mining: The explosive growth of the web presents an unprecedented opportunity for data mining. Web mining helps to discover useful knowledge from the internet. For example, top financial firms use crawlers to download shareholder meetings and annual reports to learn key company initiatives.
* Web monitoring. The crawlers help to monitor copyright and trademark infringements over the Internet. For example, Digimarc \[3] utilizes crawlers to discover pirated works and reports.

## Step 1 - Understand the problem and establish design scope

Basic algorithm:

1. Given a set of URLs, download all the web pages addressed by the URLs.
2. Extract URLs from these web pages
3. Add new URLs to the list of URLs to be downloaded. Repeat these 3 steps.

### Functional requirement

> main purpose? search engine indexing, data mining, or else? how many web pages will be collected per month? content types? HTML/PDF/images? added or edited web pages? store content? how about duplicate content?

* Search engine indexing
* 1 billion pages per month
* HTML only
* Yes, should consider the added or edited
* Store up to 5 years
* Ignore duplicate content

### Non-functional requirement:

* Scalability: The web is very large. There are billions of web pages out there. Web crawling should be extremely efficient using parallelization.
* Robustness: The web is full of traps. Bad HTML, unresponsive servers, crashes, malicious links, etc., are all common. The crawler must handle all those edge cases.
* Politeness: The crawler should not make too many requests to a website within a short time interval.
* Extensibility: The system is flexible so that minimal changes are needed to support new content types. For example, if we want to crawl image files in the future, we should not need to redesign the entire system.

### Back-of-the-envelope estimation

* Assume 1 billion web pages are downloaded every month.
* QPS: 1,000,000,000 / 30 days / 24 hours / 3600 seconds = \~400 pages per second.
* Peak QPS = 2 \* QPS = 800
* Assume the average web page size is 500k.
* 1-billion-page x 500k = 500 TB storage per month. If you are unclear about digital storage units, go through “Power of 2” section in Chapter 2 again.
* Assuming data are stored for five years, 500 TB \* 12 months \* 5 years = 30 PB. A 30 PB storage is needed to store five-year content.

## Step 2 - Propose high-level design and get buy-in

<figure><img src=".gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure>

### Seed URLs

The starting points. Can be topics (shopping, sports), or popular websites.

### URL Frontier

Two types of crawl state:

* To be downloaded
* Already downloaded

URL Frontier used to store URLs to be downloaded as a FIFO queue.

### HTML Downloader

It downloads web pages from the internet that are provided by the URL Frontier.

### DNS resolver

The HTML downloader calls the DNS resolver to get the IP address of the URL.

### Content Parser

Parse and validate web pages to avoid provoking problems and wasting storage space.

> Why a separate component?

Implementing a content parser in a crawl server will slow down the crawling process

### Content Seen?

29% of the web pages have duplicated content. This component is used to dedup.

> How to compare if two web pages are the same?

Use hash values.

### Content Storage

Both disk and memory are used

* Most of the content is stored on disk because the data set is too big to fit in memory.
* Popular content is kept in memory to reduce latency.

### URL Extractor

Parses and extracts links from HTML pages

<figure><img src=".gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

### URL Filter

The URL filter excludes certain content types, file extensions, error links, and URLs in “blacklisted” sites.

### URL Seen?

“URL Seen?” helps to avoid adding the same URL multiple times as this can increase server load and cause potential infinite loops

> How to implement it?

Use a Bloom filter and a hash table.

### URL Storage

URL Storage stores already visited URLs

## Step 3 - Design deep dive

### Depth-first search (DFS) vs Breadth-first search (BFS)

DFS is usually not a good choice because the depth of DFS can be very deep.

BFS is commonly used by web crawlers and is implemented by a first-in-first-out (FIFO) queue. In a FIFO queue, URLs are dequeued in the order they are enqueued. However, this implementation has two problems:

* Most links from the same web page are linked back to the same host. Too many calls to the same host seem as “impolite.”

<figure><img src=".gitbook/assets/image (5) (1).png" alt=""><figcaption></figcaption></figure>

* BFS does not have priority, and not every page has the same level of quality and importance. We may want to prioritize URLs according to their page ranks, web traffic, update frequency, etc

### URL frontier

URL frontier to ensure politeness, URL prioritization, and freshness

#### Politeness

Politeness is to download one page at a time from the same host to avoid being "impolite" and DOS. A delay can be added between two download tasks. The politeness constraint is implemented by maintaining a mapping from website hostnames to download (worker) threads. Each downloader thread has a separate FIFO queue and only downloads URLs obtained from that queue

<figure><img src=".gitbook/assets/image (6) (1).png" alt=""><figcaption></figcaption></figure>

* Queue router: It ensures that each queue (b1, b2, … bn) only contains URLs from the same host.
* Mapping table: It maps each host to a queue.
* FIFO queues b1, b2 to bn: Each queue contains URLs from the same host.
* Queue selector: Each worker thread is mapped to a FIFO queue, and it only downloads URLs from that queue. The queue selection logic is done by the Queue selector.
* Worker thread 1 to N. A worker thread downloads web pages one by one from the same host. A delay can be added between two download tasks.

<figure><img src=".gitbook/assets/image (7) (1).png" alt=""><figcaption></figcaption></figure>

#### Priority

We prioritize URLs based on usefulness, which can be measured by PageRank \[10], website traffic, update frequency, etc.

<figure><img src=".gitbook/assets/image (8) (1).png" alt=""><figcaption></figcaption></figure>

* Prioritizer: It takes URLs as input and computes the priorities.
* Queue f1 to fn: Each queue has an assigned priority. Queues with high priority are selected with higher probability.
* Queue selector: Randomly choose a queue with a bias towards queues with higher priority.

**URL forniter design**, it contains two modules:\
• Front queues: manage prioritization\
• Back queues: manage politeness

<figure><img src=".gitbook/assets/image (9) (1).png" alt=""><figcaption></figcaption></figure>

#### Freshness

Web pages are constantly being added, deleted, and edited. A web crawler must periodically recrawl downloaded pages to keep our data set fresh. Recrawl all the URLs is timeconsuming and resource intensive. Optimize by:

* Recrawl based on the web pages’ update history.
* Prioritize URLs and recrawl important pages first and more frequently

#### Storage for URL Frontier

Hybrid approach: the majority of URLs are stored on disk, maintain buffers in memory for enqueue/dequeue operations for fast reading and writing to the disk. Data in the buffer is periodically written to the disk.

> Why not to put everything in memory?

It is neither durable nor scalable

> Why not to keep everything in the disk?

The disk is slow, and it can easily become a bottleneck for the crawl

### HTML Downloader

#### Robots.txt (Robots Exclusion Protocol)

It used by websites to communicate with crawlers. It specifies what pages crawlers are allowed to download. We should cache the results of the file periodically.

#### Performance optimization&#x20;

**Distributed crawl**

To achieve high performance, crawl jobs are distributed into multiple servers, and each server runs multiple threads. The URL space is partitioned into smaller pieces; so, each downloader is responsible for a subset of the URLs.

<figure><img src=".gitbook/assets/image (10) (1).png" alt=""><figcaption></figcaption></figure>

**Cache DNS Resolver**

Maintaining our DNS cache to avoid calling DNS frequently. DNS cache keeps the domain name to IP address mapping and is updated periodically by cron jobs.

> But why?

DNS requests might take time due to the synchronous nature of many DNS interfaces. DNS response time ranges from 10ms to 200ms. Once a request to DNS is carried out by a crawler thread, other threads are blocked until the first request is completed, so it's a bottleneck.

**Locality**

Distribute crawl servers geographically. Design locality applies to most of the system components: crawl servers, cache, queue, storage, etc.

**Short timeout**

To avoid long wait time, a maximal wait time is specified. If a host does not respond within a predefined time, the crawler will stop the job and crawl some other pages.

### Robustness

* Consistent hashing: This helps to distribute loads among downloaders. A new downloader server can be added or removed using consistent hashing.
* Save crawl states and data: To guard against failures, crawl states and data are written to a storage system. A disrupted crawl can be restarted easily by loading saved states and data.
* Exception handling: Errors are inevitable and common in a large-scale system. The crawler must handle exceptions gracefully without crashing the system.
* Data validation: This is an important measure to prevent system errors.

### Extensibility

Always design a system that is flexible enough to supportt new content types

<figure><img src=".gitbook/assets/image (12) (1).png" alt=""><figcaption></figcaption></figure>

* PNG Downloader module is plugged-in to download PNG files.
* Web Monitor module is added to monitor the web and prevent copyright and trademark infringements

### Detect and avoid problematic content

#### Redundant content

As discussed previously, nearly 30% of the web pages are duplicates. Hashes or checksums help to detect duplication

#### Spider traps

A spider trap is a web page that causes a crawler to enter an infinite loop. It can be avoided by setting a maximal length for URLs. However, no onesize-fits-all solution exists to detect spider traps. Websites containing spider traps are easy to identify due to an unusually large number of web pages discovered on such websites. It is hard to develop automatic algorithms to avoid spider traps; however, a user can manually verify and identify a spider trap, and either exclude those websites from the crawler or apply some customized URL filters.

#### Data noise

Some of the contents have little or no value, such as advertisements, code snippets, spam URLs, etc. Those contents are not useful for crawlers and should be excluded if possible.

## Step 4 - Wrap up

Additional talking points:

* Server-side rendering: Numerous websites use scripts like JavaScript, AJAX, etc to generate links on the fly. If we download and parse web pages directly, we will not be able to retrieve dynamically generated links. To solve this problem, we perform server-side rendering (also called dynamic rendering) first before parsing a page \[12].
* Filter out unwanted pages: With finite storage capacity and crawl resources, an anti-spam component is beneficial in filtering out low quality and spam pages \[13] \[14].
* Database replication and sharding: Techniques like replication and sharding are used to improve the data layer availability, scalability, and reliability.
* Horizontal scaling: For large scale crawl, hundreds or even thousands of servers are needed to perform download tasks. The key is to keep servers stateless.
* Availability, consistency, and reliability: These concepts are at the core of any large system’s success.
* Analytics: Collecting and analyzing data are important parts of any system because data is key ingredient for fine-tuning.
