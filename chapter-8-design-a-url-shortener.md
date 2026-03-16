# Chapter 8: DESIGN A URL SHORTENER

## Overview:

Designing a URL shortening service like tinyurl

## Step 1 - Understand the problem and establish design scope

> Traffic volume? How long of the shortened URL? What character are allowed? Can it be deleted or updated?

* 100 million URLs are generated per day
* As short as possible
* Combination of numbers (0-9) and characters (a-z, AZ)
* URLs cannot be deleted or updated

**Basic use cases:**

1. URL shortening: given a long URL => return a much shorter URL
2. URL redirecting: given a shorter URL => redirect to the original URL
3. High availability, scalability, and fault tolerance considerations

### Back-of-the-envelope estimation

* Write operation: 100 million URLs are generated per day.
* Write operation per second: 100 million / 24 /3600 = 1160
* Read operation: Assuming ratio of read operation to write operation is 10:1, read operation per second: 1160 \* 10 = 11,600
* Assuming the URL shortener service will run for 10 years, this means we must support 100 million \* 365 \* 10 = 365 billion records.
* Assume average URL length is 100.
* Storage requirement over 10 years: 365 billion \* 100 bytes \* 10 years = 365 TB

## Step 2 - Propose high-level design and get buy-in

### **URL shortening**

> POST api/v1/data/shorten\
> • request parameter: {longUrl: longURLString}\
> • return shortURL

Use hash function to shrot the URL. The hash function must satisfy the following requirements:

* Each longURL must be hashed to one hashValue.
* Each hashValue can be mapped back to the longURL.

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

### **URL redirecting**

> GET api/v1/shortUrl\
> • Return longURL for HTTP redirection

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

**301 VS 302**

* 301 is permanently direct. Since it is permanently redirected, the browser caches the response, and subsequent requests for the same URL will not be sent to the URL shortening service. Instead, requests are redirected to the long URL server directly
* 302 is temporary, meaning that subsequent requests for the same URL will be sent to the URL shortening service first. Then, they are redirected to the long URL server.

If the priority is to reduce the server load, use 301.

If analytics is important, use 302 as it can track click rate and source of the click more easily

> How to store the URL?

Hashtable: \<shortURL, longURL>

* Get longURL: longURL = hashTable.get(shortURL)
* Once you get the long URL, perform the URL redirect.

## Step 3 - Design deep dive

### Data model

Hashtable is good, but not feasible for real-world systems as memory resources are limited\
and expensive. A better way is to store \<shortURL, longURL> mapping in a relational database.

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

### Hash function

#### Hash value length

The hashValue consists of characters from \[0-9, a-z, A-Z], containing 10 + 26 + 26 = 62\
possible characters. The hush value length could be:

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

#### Hash + collision resolution

Well-known hash functions are: CRC32, MD5, or SHA-1

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

> &#x20;Even the shortest hash value (from CRC32) is too long (more than 7 characters). How can we make it shorter?

The first approach is to collect the first 7 characters of a hash value; however, this method\
can lead to hash collisions.&#x20;

> How to resolve hash collisions?

We can recursively append a new predefined string until no more collisions are discovered.

This method can eliminate collisions; however, it is expensive to query the database to check if a short URL exists for every request. A technique called bloom filters can improve performance. A bloom filter is a space-efficient probabilistic technique to test if an element is a member of a set.

#### Base 62 conversion

Base conversion helps to convert the same number between its different number representation systems. Base 62 conversion is used as there are 62 possible characters for hashValue

From its name, base 62 is a way of using 62 characters for encoding. The mappings are:\
0-0, ..., 9-9, 10-a, 11-b, ..., 35-z, 36-A, ..., 61-Z, where ‘a’ stands for 10, ‘Z’ stands for 61,\
etc.

* 1115710 = 2 x 62, 2 + 55 x 62, 1 + 59 x 62, 0 = \[2, 55, 59] -> \[2, T, X] in base 62
* Thus, the short URL is **https://tinyurl.com /2TX**

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

#### Comparison of the two approaches

<figure><img src=".gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

### URL shortening deep dive

<figure><img src=".gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

> How to generate a global unique ID?

Use the distributed unique ID generator. Check chapter 7

### URL redirecting deep dive

Use cache to improve read performance

<figure><img src=".gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

## Step 4 - Wrap up

Additional talking points:

* Rate limiter:&#x20;
  * What if malicious users send an overwhelmingly large number of URL shortening requests?  A rate limiter helps to filter out requests based on IP address or other filtering rules.
* Web server scaling: Use a stateless web tier to easily scale by adding or removing web servers.
* Database scaling: Database replication and sharding are common techniques.
* Analytics: Data is increasingly important for business success. Integrating an analytics solution to the URL shortener could help to answer important questions like how many people click on a link? When do they click the link? etc.
* Availability, consistency, and reliability. Check Chapter 1 to refresh your memory on these topics.
