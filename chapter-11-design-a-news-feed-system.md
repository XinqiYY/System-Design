# Chapter 11: DESIGN A NEWS FEED SYSTEM

## Overview

News feed is the constantly updating list of stories in the middle of your home page. News Feed includes status updates, photos, videos, links, app activity, and likes from people, pages, and groups that you follow on Facebook

## Step 1 - Understand the problem and establish design scope

### Functional requirement

* Mobild app and web app
* Publish and see her friend's posts
* Feed is sorted by reverse chronological order
* 5000 friends a user can have
* 10 million DAU
* Feed contains images and videos.

## Step 2 - Propose high-level design and get buy-in

* Feed publishing: when a user publishes a post, corresponding data is written into cache and database. A post is populated to her friends’ news feed.
* Newsfeed building: for simplicity, let us assume the news feed is built by aggregating friends’ posts in reverse chronological order.

### Newsfeed APIs

#### Feed publishing API

> `POST /v1/me/feed`\
> Params:
>
> * content: content is the text of the post.
> * auth\_token: it is used to authenticate API requests.

#### Newsfeed retrieval API

> `GET /v1/me/feed`\
> Params:
>
> * auth\_token: it is used to authenticate API requests.

### Feed publishing

<figure><img src=".gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

* User: a user can view news feeds on a browser or mobile app. A user makes a post with content “Hello” through API: /v1/me/feed?content=Hello\&auth\_token={auth\_token}
* Load balancer: distribute traffic to web servers.
* Web servers: web servers redirect traffic to different internal services.
* Post service: persist post in the database and cache.
* Fanout service: push new content to friends’ news feed. Newsfeed data is stored in the cache for fast retrieval.
* Notification service: inform friends that new content is available and send out push notifications.

### Newsfeed building

<figure><img src=".gitbook/assets/image (67).png" alt=""><figcaption></figcaption></figure>

* User: a user sends a request to retrieve her news feed. The request looks like this:
  * > / v1/me/feed.
* Load balancer: load balancer redirects traffic to web servers.
* Web servers: web servers route requests to newsfeed service.
* Newsfeed service: news feed service fetches news feed from the cache.
* Newsfeed cache: store news feed IDs needed to render the news feed.

## Step 3 - Design deep dive

### Feed publishing deep dive

<figure><img src=".gitbook/assets/image (68).png" alt=""><figcaption></figcaption></figure>

#### Web servers

Web servers enforce authentication and rate-limiting

#### Fanout service

Fanout is the process of delivering a post to all friends. Two types:

**Fanout on write (also called push model)**

News feed is pre-computed during write time. A new post is delivered to friends’ cache immediately after it is published.\
Pros:

* The news feed is generated in real-time and can be pushed to friends immediately.
* Fetching news feed is fast because the news feed is pre-computed during write time.

Cons:

* If a user has many friends, fetching the friend list and generating news feeds for all of\
  them are slow and time consuming. It is called hotkey problem.
* For inactive users or those rarely log in, pre-computing news feeds waste computing\
  resources.

Fanout on read (also called pull model)

The news feed is generated during read time. This is an on-demand model. Recent posts are pulled when a user loads her home page.\
Pros:

* It does not waste computing resources for inactive users or those who rarely log in
* Data is not pushed to friends, so there is no hotkey problem.

Cons:

* Fetching the news feed is slow as the news feed is not pre-computed.

> Then what should we do?

Use a hybrid approach. Fetching the news feed fast is crucial. We use a push model for the majority of users (celebrities or users who have many friends/followers), we let followers pull news content\
on-demand to avoid system overload.

**Fanout service workflow**

1. Fetch friend IDs from the graph database. Graph databases are suited for managing friend relationships and friend recommendations.
2. Get friends' info from the user cache. The system then filters out friends based on user settings.
   1. For example, if you mute someone, her posts will not show up on your news feed even though you are still friends. Another reason why posts may not show is that a user could selectively share information with specific friends or hide it from other people.&#x20;
3. Send friends list and new post ID to the message queue.&#x20;
4. Fanout workers fetch data from the message queue and store news feed data in the news feed cache. You can think of the news feed cache as a \<post\_id, user\_id> mapping table. Whenever a new post is made, it will be appended to the news feed table as shown in below. The memory consumption can become very large if we store the entire user and post objects in the cache. Thus, only IDs are stored. To keep the memory size small, we set a configurable limit. The chance of a user scrolling through thousands of posts in news feed is slim. Most users are only interested in the latest content, so the cache miss rate is low.&#x20;
5. Store \<post\_id, user\_id > in news feed cache.

<figure><img src=".gitbook/assets/image (69).png" alt=""><figcaption></figcaption></figure>

### Newsfeed retrieval deep dive

<figure><img src=".gitbook/assets/image (70).png" alt=""><figcaption></figcaption></figure>

1. A user sends a request to retrieve her news feed. The request looks like this: /v1/me/feed&#x20;
2. The load balancer redistributes requests to web servers.&#x20;
3. Web servers call the news feed service to fetch news feeds.&#x20;
4. News feed service gets a list post IDs from the news feed cache.&#x20;
5. A user’s news feed is more than just a list of feed IDs. It contains username, profile picture, post content, post image, etc. Thus, the news feed service fetches the complete user and post objects from caches (user cache and post cache) to construct the fully hydrated news feed.&#x20;
6. The fully hydrated news feed is returned in JSON format back to the client for rendering.

### Cache architecture

Cache is extremely important for a news feed system. We divide the cache tier into 5 layers

<figure><img src=".gitbook/assets/image (71).png" alt=""><figcaption></figcaption></figure>

* News Feed: It stores IDs of news feeds.
* Content: It stores every post data. Popular content is stored in hot cache.
* Social Graph: It stores user relationship data.
* Action: It stores info about whether a user liked, replied a post, or took other actions on a post.
* Counters: It stores counters for like, reply, follower, following, etc.

## Step 4 - Wrap up

May talk about:

Scaling the database:

* Vertical scaling vs Horizontal scaling
* SQL vs NoSQL
* Master-slave replication
* Read replicas
* Consistency models
* Database sharding

Other talking points:

* Keep the web tier stateless
* Cache data as much as you can
* Support multiple data centers
* Lose a couple of components with message queues
* Monitor key metrics. For instance, QPS during peak hours and latency while users are refreshing their news feed are interesting to monitor.
