# Chapter 12: DESIGN A CHAT SYSTEM

## Overview

Design a chat app like Facebook Messenger with an emphasis on the following features:

* A one-on-one chat with low delivery latency
* Small group chat (max of 100 people)
* Online presence
* Multiple device support. The same account can be logged in to multiple accounts at the same time.
* Push notifications

## Step 1 - Understand the problem and establish design scope

### Functional requirement:

* Support both 1v1 and group chat
* Both mobile app and web app.
* It support 50 million DAU
* Max 100 people per group chat
* Important feature: 1v1 chat, group chat, online indicator. Only support text message.
* Text length should be less than 100,000 chars long
* End to end encryption is not required now.
* Store the chat history forever

## Step 2 - Propose high-level design and get buy-in

> How client and servers communicate to each other?

In a chat system, clients can be either mobile applications or web applications. Clients do not communicate directly with each other. Instead, each client connects to a chat service

The chat service must support the following functions:

* Receive messages from other clients.
* Find the right recipients for each message and relay the message to the recipients.
* If a recipient is not online, hold the messages for that recipient on the server until she is online

**Relationships between clients (sender and receiver) and the chat service**

<figure><img src=".gitbook/assets/image (72).png" alt=""><figcaption></figcaption></figure>

> Since HTTP is client-initiated, how server send messages back?

### Polling

The client periodically asks the server if there are messages available, but it could be costly depending on the polling frequency.

<figure><img src=".gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

### Long polling

In long polling, a client holds the connection open until there are actually new messages available or a timeout threshold has been reached. Once the client receives new messages, it immediately sends another request to the server, restarting the process.&#x20;

<figure><img src=".gitbook/assets/image (74).png" alt=""><figcaption></figcaption></figure>

#### Drawbacks:

* Sender and receiver may not connect to the same chat server. HTTP based servers are usually stateless. If you use round robin for load balancing, the server that receives the message might not have a long-polling connection with the client who receives the message.
* A server has no good way to tell if a client is disconnected.
* It is inefficient. If a user does not chat much, long polling still makes periodic connections after timeouts.

### WebSocket

WebSocket is the most common solution for sending asynchronous updates from the server to the client. WebSocket connection is initiated by the client. It is bi-directional and persistent. It starts its life as a HTTP connection and could be “upgraded” via some well-defined handshake to a WebSocket connection. Through this persistent connection, a server could send updates to a client. WebSocket connections generally work even if a firewall is in place. This is because they use port 80 or 443 which are also used by HTTP/HTTPS connections.

<figure><img src=".gitbook/assets/image (75).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (76).png" alt=""><figcaption></figcaption></figure>

### High-level design

<figure><img src=".gitbook/assets/image (77).png" alt=""><figcaption></figcaption></figure>

#### Stateless Services

Stateless services are traditional public-facing request/response services, used to manage the login, signup, user profile, etc

> What is servive discovery?

Its primary job is to give the client a list of DNS host names of chat servers that the client could connect to.

#### Stateful Service

The service is stateful because each client maintains a persistent network connection to a chat server so it will not switch to another chat server as long as the server is still available. The service discovery coordinates closely with the chat service to avoid server overloading.

#### Third-party integration

Notifications inform users when new messages have arrived, even when the app is not running.

#### Scalability

Be aware of a single point of failure. But it's ok to start with one server, but let your interview now about it, and you will scale it later.

<figure><img src=".gitbook/assets/image (79).png" alt=""><figcaption></figcaption></figure>

The client maintains a persistent WebSocket connection to a chat server for real-time messaging.

* Chat servers facilitate message sending/receiving.
* Presence servers manage online/offline status.
* API servers handle everything, including user login, signup, change profile, etc.
* Notification servers send push notifications.
* Finally, the key-value store is used to store chat history. When an offline user comes online, she will see all her previous chat history.

#### Storage

> Relational databases or NoSQL databases?

It depends on data types and read/write patterns

Two types of data exist in a typical chat system.

1. Generic data, such as user profile, setting, user friends list. These data are stored in robust and reliable relational databases. Use replication and sharding for availability and scalability.
2. Chat history data. It is important to understand the read/write pattern.
   1. Data is enormous for chat systems. A previous study reveals that Facebook messenger and Whatsapp process 60 billion messages a day.
   2. Only recent chats are accessed frequently. Users do not usually look up for old chats.
   3. Although very recent chat history is viewed in most cases, users might use features that require random access of data, such as search, view your mentions, jump to specific messages, etc. These cases should be supported by the data access layer.
   4. The read to write ratio is about 1:1 for 1 on 1 chat apps.

Use key-value stores for chat history because:

* Key-value stores allow easy horizontal scaling.
* Key-value stores provide very low latency to access data.
* Relational databases do not handle long tail of data well. When the indexes grow large, random access is expensive.
* Key-value stores are adopted by other proven reliable chat applications. For example, both Facebook messenger and Discord use key-value stores. Facebook messenger uses HBase, and Discord uses Cassandra

#### Data models

**Message table for 1 on 1 chat**

The primary key is message\_id, which helps to decide message sequence. We cannot rely on created\_at to decide the message sequence because two messages can be created at the same time.

<figure><img src=".gitbook/assets/image (80).png" alt=""><figcaption></figcaption></figure>

**Message table for group chat**

Primary key is (channel\_id, message\_id). Channel and group represent the same meaning here. channel\_id is the partition key because all queries in a group chat operate in a channel.

<figure><img src=".gitbook/assets/image (81).png" alt=""><figcaption></figcaption></figure>

#### Message ID

Message\_id carries the responsibility of ensuring the order of messages. To ascertain the order of messages, message\_id must satisfy the following two requirements:

* IDs must be unique.
* IDs should be sortable by time, meaning new rows have higher IDs than old ones.

> How to generate MessageID?

1. Use "auto\_increment” keyword in MySql, but NoSQL doesn't provide it
2. Use a global 64-bit sequence number generator like Snowflake
3. Use a local sequence number generator. Local means IDs are only unique within a group. The reason why local IDs work is that maintaining message sequence within one-on-one channel or a group channel is sufficient. This approach is easier to implement in comparison to the global ID implementation

## Step 3 - Design deep dive

### Service discovery

The primary role of service discovery is to recommend the best chat server for a client based on the criteria like geographical location, server capacity, etc. Apache Zookeeper \[7] is a popular open-source solution for service discovery. It registers all the available chat servers and picks the best chat server for a client based on predefined criteria.

<figure><img src=".gitbook/assets/image (82).png" alt=""><figcaption></figcaption></figure>

1. User A tries to log in to the app.
2. The load balancer sends the login request to API servers.&#x20;
3. After the backend authenticates the user, service discovery finds the best chat server for User A. In this example, server 2 is chosen and the server info is returned back to User A.&#x20;
4. User A connects to chat server 2 through WebSocket.

### Message flows

#### 1 on 1 chat flow

<figure><img src=".gitbook/assets/image (83).png" alt=""><figcaption></figcaption></figure>

1. User A sends a chat message to Chat server 1.&#x20;
2. Chat server 1 obtains a message ID from the ID generator.&#x20;
3. Chat server 1 sends the message to the message sync queue.&#x20;
4. The message is stored in a key-value store.&#x20;
5. a: If User B is online, the message is forwarded to Chat server 2 where User B is connected.&#x20;
6. b: If User B is offline, a push notification is sent from push notification (PN) servers.&#x20;
7. Chat server 2 forwards the message to User B. There is a persistent WebSocket connection between User B and Chat server 2.

#### Message synchronization across multiple devices

<figure><img src=".gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>

User A has two devices: a phone and a laptop. When User A logs in to the chat app with her phone, it establishes a WebSocket connection with Chat server 1. Similarly, there is a connection between the laptop and Chat server 1. Each device maintains a variable called cur\_max\_message\_id, which keeps track of the latest message ID on the device.&#x20;

Messages that satisfy the following two conditions are considered as news messages:

* The recipient ID is equal to the currently logged-in user ID.
* Message ID in the key-value store is larger than cur\_max\_message\_id .&#x20;

With distinct cur\_max\_message\_id on each device, message synchronization is easy as each device can get new messages from the KV store.

#### Small group chat flow

**From sender side**

<figure><img src=".gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

The message from User A is copied to each group member’s message sync queue, but it is only good for small group chat (500, WeChat is using this approach, because: &#x20;

* It simplifies message sync flow as each client only needs to check its own inbox to get new messages.\
  • When the group number is small, storing a copy in each recipient’s inbox is not too expensive.

**From the recipient side:**

On the recipient side, a recipient can receive messages from multiple users. Each recipient\
has an inbox (message sync queue) that contains messages from different senders.

<figure><img src=".gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

### Online presence

The presence servers are responsible for managing online status and communicating with clients through WebSocket. There are a few flows that will trigger online status change.

#### User login&#x20;

The user login flow is explained in the “Service Discovery” section. After a WebSocket connection is built between the client and the real-time service, user A’s online status and last\_active\_at timestamp are saved in the KV store. Presence indicator shows the user is online after she logs in.

<figure><img src=".gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

#### User logout

When a user logs out,&#x20;

<figure><img src=".gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure>

The online status is changed to offline in the KV store.

#### User disconnection

> Due to poor internet connection, if we just mark the user as offline and online, it will make the presence indicator change too offten, resulting in poor user experience. How to fix it?

Heartbeat mechanism. Periodically, an online client sends a heartbeat event to presence servers. If presence servers receive a heartbeat event within a certain time, say x seconds from the client, a user is considered as online. Otherwise, it is offline.

<figure><img src=".gitbook/assets/image (90).png" alt=""><figcaption></figcaption></figure>

#### Online status fanout

> How do user A’s friends know about the status changes?

Presence servers use a publish-subscribe model, in which each friend pair maintains a channel.

When User A’s online status changes, it publishes the event to three channels, channel A-B, A-C, and A-D. Those three channels are subscribed by User B, C, and D, respectively. Thus, it is easy for friends to get online status updates. The communication between clients and servers is through real-time WebSocket.

<figure><img src=".gitbook/assets/image (91).png" alt=""><figcaption></figcaption></figure>

> But it only effective for a small user group for 500 people but expensive and time consumming for large group. What if the group has 100,000 members?

Fetch online status only when a user enters a group or manually refreshes the friend list.

## Step 4 - Wrap up

Additional talking points:

* Extend the chat app to support media files such as photos and videos. Media files are significantly larger than text in size. Compression, cloud storage, and thumbnails are interesting topics to talk about.
* End-to-end encryption. Whatsapp supports end-to-end encryption for messages. Only the sender and the recipient can read messages. Interested readers should refer to the article in the reference materials \[9].
* Caching messages on the client-side is effective to reduce the data transfer between the client and server.
* Improve load time. Slack built a geographically distributed network to cache users’ data, channels, etc. for better load time \[10].
* Error handling:
  * The chat server error. There might be hundreds of thousands, or even more persistent connections to a chat server. If a chat server goes offline, service discovery (Zookeeper) will provide a new chat server for clients to establish new connections with.
  * Message resent mechanism. Retry and queueing are common techniques for resending messages.
