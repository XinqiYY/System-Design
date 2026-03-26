# Chapter 15: DESIGN GOOGLE DRIVE

## Step 1 - Understand the problem and establish design scope

### Functional requirement

* Upload and download files, file sync, and notifications
* Both mobile app and web app
* Support any file formats
* File in the storage must be encrypted
* File size must be <= 10 GB
* 10m DAU

In this chapter, we focus on add files, sync files, file versions, share files, and notification

### Non-functional requirement

* Reliability. Reliability is extremely important for a storage system. Data loss is unacceptable.
* Fast sync speed. If file sync takes too much time, users will become impatient and abandon the product.
* Bandwidth usage. If a product takes a lot of unnecessary network bandwidth, users will be unhappy, especially when they are on a mobile data plan.
* Scalability. The system should be able to handle high volumes of traffic.
* High availability. Users should still be able to use the system when some servers are offline, slowed down, or have unexpected network errors.

### Back of the envelope estimation

* Assume the application has 50 million signed up users and 10 million DAU.
* Users get 10 GB free space.
* Assume users upload 2 files per day. The average file size is 500 KB.
* 1:1 read to write ratio.
* Total space allocated: 50 million \* 10 GB = 500 Petabyte
* QPS for upload API: 10 million \* 2 uploads / 24 hours / 3600 seconds = \~ 240
* Peak QPS = QPS \* 2 = 480

## Step 2 - Propose high-level design and get buy-in

### APIs

We primarilyprimary need 3 APIs: upload a file, download a file, and get file revisions.

#### Upload a file to Google Drive

Two types of uploads are supported:

* Simple upload. Use this upload type when the file size is small.
* Resumable upload. Use this upload type when the file size is large and there is high chance of network interruption.

Example:

> https://api.example.com/files/upload?uploadType=resumable\
> Params:
>
> * uploadType=resumable
> * data: Local file to be uploaded.

A resumable upload is achieved by the following 3 steps \[2]:

* Send the initial request to retrieve the resumable URL.
* Upload the data and monitor upload state.
* If upload is disturbed, resume the upload.

#### Download a file from Google Drive

> Example API: https://api.example.com/files/download\
> Params:
>
> * path: download file path.
>
> Example params:\
> &#x20;     {\
> &#x20;           "path": "/recipes/soup/best\_soup.txt"\
> &#x20;     }

#### Get file revisions

> Example API: https://api.example.com/files/list\_revisions\
> Params:
>
> * path: The path to the file you want to get the revision history.
> * limit: The maximum number of revisions to return.
>
> Example params:\
> &#x20;     {\
> &#x20;           "path": "/recipes/soup/best\_soup.txt",\
> &#x20;           "limit": 20\
> &#x20;     }

All the APIs require user authentication and use HTTPS. Secure Sockets Layer (SSL) protects data transfer between the client and backend servers.

### Move away from a single server

Amazon S3 supports same-region and cross-region replication. data can be replicated on the same-region (left side) and cross-region (right side). Redundant files are stored in multiple regions to guard against data loss and ensure availability. A bucket is like a folder in a file system.

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

* Load balancer: Evenly distribute network traffic.
  * if a web server goes down, it will redistribute the traffic.
* Web servers: After a load balancer is added, more web servers can be added/removed easily, depending on the traffic load.
* Metadata database: Move the database out of the server to avoid single point of failure. In the meantime, set up data replication and sharding to meet the availability and scalability requirements.
* File storage: Amazon S3 is used for file storage. To ensure availability and durability, files are replicated in two separate geographical regions.

### Sync conflicts

System can present both copies of the same file: user 2’s local copy and the latest version from the server (Figure 15-9). User 2 has the option to merge both files or override one version with the other

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

> reference material \[4] \[5]

### High-level design

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

* User: Endpoint
* Block servers: Block servers upload blocks to cloud storage. Block storage, referred to as block-level storage, is a technology to store data files on cloud-based environments. A file can be split into several blocks, each with a unique hash value, stored in our metadata database. Each block is treated as an independent object and stored in our storage system (S3). To reconstruct a file, blocks are joined in a particular order. As for the block size, we use Dropbox as a reference: it sets the maximal size of a block to 4MB \[6].
* Cloud storage: A file is split into smaller blocks and stored in cloud storage.
* Cold storage: Storing inactive data, meaning files are not accessed for a long time.
* Load balancer: Evenly distributes requests among API servers.
* API servers: Support everything other than the uploading flow. API servers are used for user authentication, managing user profile, updating file metadata, etc.
* Metadata database: It stores metadata of users, files, blocks, versions, etc. Please note that files are stored in the cloud and the metadata database only contains metadata.
* Metadata cache: Some of the metadata are cached for fast retrieval.
* Notification service: Notification service notifies relevant clients when a file is added/edited/removed elsewhere so they can pull the latest changes.
* Offline backup queue: If a client is offline and cannot pull the latest file changes, the offline backup queue stores the info so changes will be synced when the client is online. See more details in the deep dive.

## Step 3 - Design deep dive

### Block servers

> How to upload large files?

Block servers allow us to save network traffic by providing delta sync and compression

* Delta sync. When a file is modified, only modified blocks are synced instead of the whole file using a sync algorithm \[7] \[8].
* Compression. Applying compression on blocks can significantly reduce the data size. Thus, blocks are compressed using compression algorithms depending on file types. For example, gzip and bzip2 are used to compress text files. Different compression algorithms are needed to compress images and videos

Block servers process files passed from clients by splitting a file into blocks, compressing each block, and encrypting it. Instead of uploading the whole file to the storage system, only modified blocks are transferred.

<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

* A file is split into smaller blocks.
* Each block is compressed using compression algorithms.
* To ensure security, each block is encrypted before it is sent to cloud storage.
* Blocks are uploaded to the cloud storage.

#### Illustrates delta sync

Highlighted blocks “block 2” and “block 5” represent changed blocks. Using delta sync, only those two blocks are uploaded to the cloud storage

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

### High consistency requirement

Differently by different clients at the same time is unacceptable. The system needs to provide strong consistency for the metadata cache and database layers.&#x20;

Memory caches adopt an eventual consistency model by default. To achieve strong consistency, we must ensure:

* Data in cache replicas and the master is consistent.
* Invalidate caches on database write to ensure that the cache and database hold the same value.

> Note: Achieving strong consistency in a relational database is easy because it maintains the ACID (Atomicity, Consistency, Isolation, Durability) properties \[9]. However, NoSQL databases do not support ACID properties by default. ACID properties must be programmatically incorporated in synchronization logic. In our design, we choose relational databases because the ACID is natively supported.

### Metadata database

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

* User: The user table contains basic information about the user such as username, email, profile photo, etc.
* Device: Device table stores device info. Push\_id is used for sending and receiving mobile push notifications. Please note a user can have multiple devices.
* Namespace: A namespace is the root directory of a user.
* File: File table stores everything related to the latest file.
* File\_version: It stores version history of a file.&#x20;
  * Existing rows are read-only to keep the integrity of the file revision history.
* Block: It stores everything related to a file block. A file of any version can be reconstructed by joining all the blocks in the correct order

## Upload flow

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

Two requests are sent in parallel: add file metadata and upload the file to cloud storage. Both requests originate from client 1.

* Add file metadata.
  * Client 1 sends a request to add the metadata of the new file.
  * Store the new file metadata in metadata DB and change the file upload status to “pending.”
  * Notify the notification service that a new file is being added.
  * The notification service notifies relevant clients (client 2) that a file is being uploaded.
* Upload files to cloud storage.&#x20;
  * 2.1 Client 1 uploads the content of the file to block servers.&#x20;
  * 2.2 Block servers chunk the files into blocks, compress, encrypt the blocks, and upload them to cloud storage.&#x20;
  * 2.3 Once the file is uploaded, cloud storage triggers upload completion callback. The request is sent to API servers.&#x20;
  * 2.4 File status changed to “uploaded” in Metadata DB.&#x20;
  * 2.5 Notify the notification service that a file status is changed to “uploaded.”&#x20;
  * 2.6 The notification service notifies relevant clients (client 2) that a file is fully uploaded. When a file is edited, the flow is similar, so we will not repeat it.

### Download flow

> Download flow is triggered when a file is added or edited elsewhere. How does a client know if a file is added or edited by another client?

* If client A is online while a file is changed by another client, notification service will inform client A that changes are made somewhere so it needs to pull the latest data.
* If client A is offline while a file is changed by another client, data will be saved to the cache. When the offline client is online again, it pulls the latest changes.

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

1. The notification service informs client 2 that a file has been changed somewhere else.&#x20;
2. Once client 2 knows that new updates are available, it sends a request to fetch metadata.&#x20;
3. API servers call the metadata DB to fetch metadata of the changes.&#x20;
4. Metadata is returned to the API servers.&#x20;
5. Client 2 gets the metadata.&#x20;
6. Once the client receives the metadata, it sends requests to block servers to download blocks.
7. Block servers first download blocks from cloud storage.&#x20;
8. Cloud storage returns blocks to the block servers.&#x20;
9. Client 2 downloads all the new blocks to reconstruct the file.

### Notification service

> Which transfered model to use?

Use long polling. Communication for the notification service is not bi-directional. Only real-time needs WebSocket.

### Save storage space

To support file version history and ensure reliability, multiple versions of the same file are stored across multiple data centers. Storage space can be filled up quickly with frequent backups of all file revisions. Three techniques are proposed to reduce storage costs:

* De-duplicate data blocks. Two blocks are identical if they have the same hash value.
* Adopt an intelligent data backup strategy. Two optimization strategies can be applied:
  * Set a limit: the oldest version will be replaced with the new version if the limit is reached
  * Keep valuable versions only
    * Limit the number of saved versions, and give more weight to recent versions. Experimentation is helpful to figure out the optimal number of versions to save.
* Moving infrequently used data to cold storage. Amazon S3 glacier \[11] is much cheaper than S3.

### Failure Handling

* Load balancer failure
  * If a load balancer fails, the secondary would become active and pick up the traffic. Load balancers usually monitor each other using a heartbeat, a periodic signal sent between load balancers. A load balancer is considered as failed if it has not sent a heartbeat for some time.
* Block server failure
  * Other servers should pick up unfinished or pending jobs.
* Cloud storage failure
  * S3 buckets are replicated multiple times in different regions. If files are not available in one region, they can be fetched from different regions.
* API server failure
  * It is a stateless service. If an API server fails, the traffic is redirected to other API servers by a load balancer.
* Metadata cache failure
  * Metadata cache servers are replicated multiple times. If one node goes down, you can still access other nodes to fetch data. We will bring up a new cache server to replace the failed one.
* Metadata DB failure.
  * Master down: If the master is down, promote one of the slaves to act as a new master and bring up a new slave node.
  * Slave down: If a slave is down, you can use another slave for read operations and bring another database server to replace the failed one.
* Notification service failure
  * Every online user keeps a long poll connection with the notification server. Thus, each notification server is connected with many users. According to the Dropbox talk in 2012 \[6], over 1 million connections are open per machine. If a server goes down, all the long poll connections are lost so clients must reconnect to a different server. Even though one server can keep many open connections, it cannot reconnect all the lost connections at once. Reconnecting with all the lost clients is a relatively slow process.
* Offline backup queue failure
  * Queues are replicated multiple times. If one queue fails, consumers of the queue may need to re-subscribe to the backup queue.

## Step 4 - Wrap up

Additional talk points:

1. Upload files directly to cloud storage, which is faster. However, a few drawbacks:
   1. The same chunking, compression, and encryption logic must be implemented on different platforms (iOS, Android, Web). It is error-prone and requires a lot of engineering effort. In our design, all those logics are implemented in a centralized place: block servers.
   2. As a client can easily be hacked or manipulated, implementing encryption logic on the client side is not ideal.
2. Use the presence service for online/offline, which can easily be integrated with other services.
