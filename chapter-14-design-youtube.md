# Chapter 14: DESIGN YOUTUBE

## Step 1 - Understand the problem and establish design scope

### Functional requirement

* Feature: upload and watch video
* Support mobile apps, web browsers, and smart TV
* 5 million DAU
* Spend on the product for 30 minutes average daily
* Support international users
* Support video resolutions and formats
* Encryption required
* Support small and medium sized videos, maximum allowed is 1GB
* Yes, leverage existing cloud infrasture is allowed.

### Non-functional requirement

* Ability to upload videos fast
* Smooth video streaming
* Ability to change video quality
* Low infrastructure costHigh availability, scalability, and reliability requirements
* Clients supported: mobile apps, web browser, and smart TV

### Back of the envelope estimation

* Assume the product has 5 million daily active users (DAU).
* Users watch 5 videos per day.
* 10% of users upload 1 video per day.
* Assume the average video size is 300 MB.
* Total daily storage space needed: 5 million \* 10% \* 300 MB = 150TB
* CDN cost.
  * When cloud CDN serves a video, you are charged for data transferred out of the CDN.
  * Let us use Amazon’s CDN CloudFront for cost estimation (Figure 14-2) \[3]. Assume 100% of traffic is served from the United States. The average cost per GB is $0.02. For simplicity, we only calculate the cost of video streaming.
  * 5 million \* 5 videos \* 0.3GB \* $0.02 = $150,000 per day

## Step 2 - Propose high-level design and get buy-in

At the high-level, the system comprises three components

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

* **Client**: You can watch YouTube on your computer, mobile phone, and smartTV.
* **CDN**: Videos are stored in CDN. When you press play, a video is streamed from the CDN.
* **API servers**: Everything else except video streaming goes through API servers. This includes feed recommendation, generating video upload URL, updating metadata database and cache, user signup, etc.

### Video uploading flow

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

* **User**: A user watches YouTube on devices such as a computer, mobile phone, or smart TV.
* **Load balancer**: A load balancer **evenly distributes requests** among API servers.
* **API servers**: All user requests go through API servers **except** video streaming.
* **Metadata DB:** Video metadata is stored in the Metadata D&#x42;**.** It is **sharded and replicated** to meet performance and high availability requirements.
* **Metadata cache**: For better performance, video metadata and user objects are **cached**.
* **Original storage**: A blob storage system is used to store original videos.&#x20;
  * A quotation in Wikipedia regarding blob storage shows that: “A Binary Large Object (BLOB) is a collection of binary data stored as a single entity in a database management system” \[6].
* **Transcoding servers**: Video transcoding is also called video encoding. It is the process of **converting a video format to other formats (MPEG, HLS, etc)**, which provide the best video streams possible for different devices and bandwidth capabilities.
* **Transcoded storage**: It is a blob storage that stores transcoded video files.
* **CDN**: Videos are cached in CDN.&#x20;
  * When you click the play button, a video is streamed from the CDN.
* **Completion queue**: It is a message queue that stores information about video transcoding completion events.
* **Completion handler**: This consists of a list of **workers** that pull event data from the completion queue and update the metadata cache and database.

#### Flow a: upload the actual video

1. Videos are uploaded to the original storage.
2. Transcoding servers fetch videos from the original storage and start transcoding.
3. Once transcoding is complete, the following two steps are executed in parallel:&#x20;
   1. 3a. Transcoded videos are sent to transcoded storage.&#x20;
   2. 3b. Transcoding completion events are queued in the completion queue.&#x20;
      1. 3a.1. Transcoded videos are distributed to CDN.&#x20;
      2. 3b.1. Completion handler contains a bunch of workers that continuously pull event data from the queue.&#x20;
   3. 3b.1.a. and 3b.1.b. Completion handler updates the metadata database and cache when video transcoding is complete.&#x20;
4. API servers inform the client that the video is successfully uploaded and is ready for streaming.

#### Update video metadata

Metadata contains information about video URL, size, resolution, format, user info, etc.

While a file is being uploaded to the original storage, the client in parallel sends a request to update the video metadata. The request contains video metadata, including file name, size, format, etc. API servers update the metadata cache and database

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

### Video streaming flow

Whenever you watch a video on YouTube, it usually starts streaming immediately instead of downloading.

Streaming means your device loads a little bit of data at a time to continuously receive video streams from remote source videos.

#### Streaming protocol

This is a standardized way to control data transfer for video streaming. Popular streaming protocols are:

* MPEG–DASH. MPEG stands for “Moving Picture Experts Group” and DASH stands for "Dynamic Adaptive Streaming over HTTP".
* Apple HLS. HLS stands for “HTTP Live Streaming”.
* Microsoft Smooth Streaming.
* Adobe HTTP Dynamic Streaming (HDS).

Different streaming protocols support different video encodings and playback players. To learn more about streaming protocols, here is an excellent article \[7]

## Step 3 - Design deep dive

### Video transcoding

> Why we need to transcode the video?

If you want the video to be played smoothly on other devices, the video must be encoded into compatible bitrates and formats.&#x20;

* Bitrate is the rate at which bits are processed over time. A higher bitrate generally means higher video quality. High bitrate streams need more processing power and fast internet speed.

> Why the video transcoding is important?

* Raw video consumes large amounts of storage space.&#x20;
  * An hour-long high definition video recorded at 60 frames per second can take up a few hundred GB of space.
* Many devices and browsers only support certain types of video formats. Thus, it is important to encode a video to different formats for compatibility reasons.
* To ensure users watch high-quality videos while maintaining smooth playback, it is a good idea to deliver higher resolution video to users who have high network bandwidth and lower resolution video to users who have low bandwidth.
* Network conditions can change, especially on mobile devices. To ensure a video is played continuously, switching video quality automatically or manually based on network conditions is essential for smooth user experience.

Encoding formats contain two parts:

* Container: This is like a basket that contains the video file, audio, and metadata. You can tell the container format by the file extension, such as .avi, .mov, or .mp4.
* Codecs: These are compression and decompression algorithms aim to reduce the video size while preserving the video quality. The most used video codecs are H.264, VP9, and HEVC.

### Directed acyclic graph (DAG) model

<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

The original video is split into video, audio, and metadata. Here are some of the tasks that can be applied to a video file:

* **Inspection**: Make sure the videos are high quality and not malformed.
* **Video encodings**: Videos are converted to support different resolutions, codecs, bitrates, etc.&#x20;
  *

      <figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>
* **Thumbnail**. Thumbnails can either be uploaded by a user or automatically generated by the system.
* **Watermark**: An image overlay on top of your video contains identifying information about your video.

### Video transcoding architecture

The proposed video transcoding architecture that leverages cloud services

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

#### Preprocessor

The preprocessor has 4 responsibilities:

1. Video splitting. Video stream is split or further split into smaller Group of Pictures (GOP) alignment. GOP is a group/chunk of frames arranged in a specific order. Each chunk is an independently playable unit, usually a few seconds in length.
2. Some old mobile devices or browsers might not support video splitting. Preprocessor split videos by GOP alignment for old clients.
3. DAG generation. The processor generates a DAG based on the configuration files that client programmers write. Example below represents a graph that has 2 nodes and 1 edge:
   1.

       <figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>
4. Cache data. The preprocessor is a cache for segmented videos. For better reliability, the preprocessor stores GOPs and metadata in temporary storage. If video encoding fails, the system could use persisted data for retry operations.

#### DAG scheduler

The DAG scheduler splits a DAG graph into stages of tasks and puts them in the task queue in the resource manager.

The original video is split into three stages: Stage 1: video, audio, and metadata. The video file is further split into two tasks in stage 2: video encoding and thumbnail. The audio file requires audio encoding as part of the stage 2 tasks.

<figure><img src=".gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

#### Resource manager

The resource manager manages the efficiency of resource allocation. It contains 3 queues and a task scheduler.

* Task queue: It is a priority queue that contains tasks to be executed.
* Worker queue: It is a priority queue that contains worker utilization info.
* Running queue: It contains info about the currently running tasks and workers running the tasks.
* Task scheduler: It picks the optimal task/worker, and instructs the chosen task worker to execute the job.

<figure><img src=".gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

The resource manager works as follows:

* The task scheduler gets the highest priority task from the task queue.
* The task scheduler gets the optimal task worker to run the task from the worker queue.
* The task scheduler instructs the chosen task worker to run the task.
* The task scheduler binds the task/worker info and puts it in the running queue.
* The task scheduler removes the job from the running queue once the job is done.

#### Task workers

Task workers run the tasks that are defined in the DAG. Different task workers may run different tasks

<figure><img src=".gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

#### Temporary storage

Metadata is frequently accessed by workers, and the data size is usually small. Thus, caching metadata in memory is a good idea. For video or audio data, we put them in blob storage. Data in temporary storage is freed up once the corresponding video processing is complete.

#### Encoded video

Encoded video is the final output of the encoding pipeline, such as _funny\_720p.mp4_.

### System optimizations

#### Speed optimization: parallelize video uploading

Uploading a video as a whole unit is inefficient. We can split a video into smaller chunks by GOP alignment. This allows fast resumable uploads when the previous upload failed. The job of splitting a video file by GOP can be implemented by the client to improve the upload speed

<figure><img src=".gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

#### Speed optimization: place upload centers close to users

Another way to improve the upload speed is by setting up multiple upload centers across the\
globe. Such use CDN as upload centers.

#### Speed optimization: parallelism everywhere

Build a loosely coupled system and enable high parallelism. For example, the flow of how a video is transferred from original storage to the CDN has a highly dependent flow.&#x20;

<figure><img src=".gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

To make the system more loosely coupled, we introduced message queues

<figure><img src=".gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

* Before the message queue is introduced, the encoding module must wait for the output of the download module.
* After the message queue is introduced, the encoding module does not need to wait for the output of the download module anymore. If there are events in the message queue, the encoding module can execute those jobs in parallel.

#### Safety optimization: pre-signed upload URL

<figure><img src=".gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

1. The client makes a HTTP request to API servers to fetch the pre-signed URL, which gives the access permission to the object identified in the URL. The term pre-signed URL is used by uploading files to Amazon S3. Other cloud service providers might use a different name. For instance, Microsoft Azure blob storage supports the same feature, but call it “Shared Access Signature” \[10].&#x20;
2. API servers respond with a pre-signed URL.&#x20;
3. Once the client receives the response, it uploads the video using the pre-signed URL.

#### Safety optimization: protect your videos

To protect copyrighted videos, we can support:

* **Digital rights management (DRM) systems**: Three major DRM systems are Apple FairPlay, Google Widevine, and Microsoft PlayReady.
* **AES encryption**: You can encrypt a video and configure an authorization policy. The encrypted video will be decrypted upon playback. This ensures that only authorized users can watch an encrypted video.
* **Visual watermarking**: This is an image overlay on top of your video that contains identifying information for your video. It can be your company logo or company name

#### Cost-saving optimization

> CDN is expensive, how to reduce the cost?

1. Only serve the most popular videos from CDN and other videos from our high-capacity servers
   1.

       <figure><img src=".gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>
2. For less popular content, we may not need to store many encoded video versions. Short videos can be encoded on-demand.
3. No need to distribute for some videos are popular only in certain regions.
4. Build your own CDN like Netflix and partner with Internet Service Providers (ISPs). Building your CDN is a giant project; however, this could make sense for large streaming companies. An ISP can be Comcast, AT\&T, Verizon, or other internet providers. ISPs are located all around the world and are close to users. By partnering with ISPs, you can improve the viewing experience and reduce the bandwidth charges

### Error handling

* Recoverable error. For recoverable errors, such as a video segment failing to transcode, the general idea is to retry the operation a few times. If the task continues to fail and the system believes it is not recoverable, it returns a proper error code to the client.
* Non-recoverable error. For non-recoverable errors, such as malformed video format, the system stops the running tasks associated with the video and returns the proper error code to the client.

Typical errors for each system component are covered by the following playbook:

* Upload error: retry a few times.
* Split video error: if older versions of clients cannot split videos by GOP alignment, the entire video is passed to the server. The job of splitting videos is done on the server-side.
* Transcoding error: retry.
* Preprocessor error: regenerate DAG diagram.
* DAG scheduler error: reschedule a task.
* Resource manager queue down: use a replica.
* Task worker down: retry the task on a new worker.
* API server down: API servers are stateless so requests will be directed to a different API server.
* Metadata cache server down: data is replicated multiple times. If one node goes down, you can still access other nodes to fetch data. We can bring up a new cache server to replace the dead one.
  * Metadata DB server down:
    * Master is down. If the master is down, promote one of the slaves to act as the new master.
    * Slave is down. If a slave goes down, you can use another slave for reads and bring up another database server to replace the dead one.

## Step 4 - Wrap up

Additional points:

* Scale the API tier: Because API servers are stateless, it is easy to scale API tier horizontally.
* Scale the database: You can talk about database replication and sharding.
* Live streaming: It refers to the process of how a video is recorded and broadcasted in real time. Although our system is not designed specifically for live streaming, live streaming and non-live streaming have some similarities: both require uploading, encoding, and streaming. The notable differences are:
  * Live streaming has a higher latency requirement, so it might need a different streaming protocol.
  * Live streaming has a lower requirement for parallelism because small chunks of data are already processed in real-time.
  * Live streaming requires different sets of error handling. Any error handling that takes too much time is not acceptable.
* Video takedowns: Videos that violate copyrights, pornography, or other illegal acts shall be removed. Some can be discovered by the system during the upload process, while others might be discovered through user flagging.
