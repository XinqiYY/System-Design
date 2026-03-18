# Chapter 10: DESIGN A NOTIFICATION SYSTEM

## Overview

Three types of notification formats are: mobile push notification, SMS message, and Email.

## Step 1 - Understand the problem and establish design scope

> What types? A real-time system? Supported devices? What triggers notifications? Will use be able to opt-out? How many notifications are sent out each day?

### Functional requirement:

* All three types
* Soft real-time, user receive notifications ASAP, but slight delay is acceptable
* iOS devices, android devices, and laptop/desktop
* Triggered by client applications. They can also be scheduled on the server-side.
* Yes, users who choose to opt-out will no longer receive notifications.
* 10 million mobile push notifications, 1 million SMS messages, and 5 million emails.

## Step 2 - Propose high-level design and get buy-in

### Different types of notifications

#### iOS push notification

<figure><img src=".gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

*   Provider. A provider builds and sends notification requests to Apple Push Notification Service (APNS). To construct a push notification, the provider provides the following data:

    * Device token: This is a unique identifier used for sending push notifications.
    * Payload:

    <figure><img src=".gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>
* APNS: This is a remote service provided by Apple to propagate push notifications to iOS devices.
* iOS Device: It is the end client that receives push notifications

#### Android push notification

Instead of using APNs, Firebase Cloud Messaging (FCM) is used in Android devices.

<figure><img src=".gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

#### SMS message

For SMS messages, third party SMS services like Twilio \[1], Nexmo \[2], and many others are commonly used. Most of them are commercial services.

<figure><img src=".gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

#### Email

Although companies can set up their own email servers, many of them opt for commercial email services. Sendgrid and Mailchimp are among the most popular email services, which offer a better delivery rate and data analytics.

<figure><img src=".gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

### Contact info gathering flow

To send notifications, we need to gather mobile device tokens, phone numbers, or email addresses. When a user installs our app or signs up for the first time, API servers collect user contact info and store it in the database.

<figure><img src=".gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

Email addresses and phone numbers are stored in the user table, whereas device tokens are stored in the device table. A user can have multiple devices, indicating that a push notification can be sent to all the user devices.

<figure><img src=".gitbook/assets/image (58).png" alt=""><figcaption></figcaption></figure>

### Notification sending/receiving flow

#### High-level design

<figure><img src=".gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

#### Service 1 to N

A service can be a microservice, a cron job, or a distributed system that\
triggers notification sending events. For example, a billing service sends emails to remind\
customers of their overdue payment, or a shopping website tells customers that their packages will\
be delivered tomorrow via SMS messages.

#### Notification system

The notification system is the centerpiece of sending/receiving notifications. It provides APIs for services 1 to N, and builds notification payloads for third-party services.

#### Third-party services

Need to pay extra attention to extensibility, which means a flexible system that can easily plug in or plug out a third-party service.

> Note: third-party service might be unavailable in new markets or in the future. For instance, FCM is unavailable in China. Thus, alternative third-party services such as Jpush, PushY, etc are used there.

#### iOS, Android, SMS, Email

Three problems are identified in this design:

* Single point of failure (SPOF): A single notification server means SPOF.
* Hard to scale: Everything in one server is challenging to scale databases, caches, and different notification processing components independently.
* Performance bottleneck: Processing and sending notifications can be resource-intensive. For example, constructing HTML pages and waiting for responses from third party services could take time. Handling everything in one system can result in the system overload, especially during peak hours.

#### High-level design (improved)

After enumerating challenges in the initial design, we improved the design as listed below:

* Move the database and cache out of the notification server.
* Add more notification servers and set up automatic horizontal scaling.
* Introduce message queues to decouple the system components.

<figure><img src=".gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

#### Notification servers

* Provide APIs for services to send notifications. Those APIs are only accessible internally or by verified clients to prevent spam.
* Carry out basic validations to verify emails, phone numbers, etc.
* Query the database or cache to fetch data needed to render a notification.
* Put notification data to message queues for parallel processing

Example of the API to send an email:\
`POST https://api.example.com/v/sms/send`\
Request body:&#x20;

<figure><img src=".gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

**Cache**: User info, device info, and notification templates are cached.&#x20;

**DB: It stores data about users, notifications, settings, etc.**&#x20;

Message queues: Decoupled. Message queues serve as buffers when high volumes of notifications are to be sent out. Each notification type is assigned to a distinct message queue, so an outage in one third-party service will not affect other notification types. Workers:&#x20;

**Workers** are a list of servers that pull notification events from message queues and send them to the corresponding third-party services.&#x20;

**Third-party services**: Already explained in the initial design.&#x20;

**iOS, Android, SMS, Email**: Already explained in the initial design.&#x20;

Next, let us examine how every component works together to send a notification:

1. A service calls APIs provided by notification servers to send notifications.
2. Notification servers fetch metadata such as user info, device token, and notification setting from the cache or database.
3. A notification event is sent to the corresponding queue for processing. For instance, an iOS push notification event is sent to the iOS PN queue.
4. Workers pull notification events from message queues.
5. Workers send notifications to third-party services.
6. Third-party services send notifications to user devices.

## Step 3 - Design deep dive

### Reliability

> How to prevent data loss?

Notifications can usually be delayed or re-ordered, but never lost. To satisfy this requirement, the notification system persists notification data in a database and implements a retry mechanism.

<figure><img src=".gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>

> Will recipients receive a notification exactly once?

No, although notification is delivered exactly once most of the time, it still can't prevent duplication; it needs to be reduced via a dedupe mechanism and handled carefully in each failure case.

Example:

When a notification event first arrives, we check if it is seen before by checking the event ID. If it is seen before, it is discarded. Otherwise, we will send out the notification.

### Additional components and considerations

#### Notification template

Notification templates are introduced to avoid building every notification from scratch

#### Notification setting

Websites and apps should give users control over notification settings. Before any notification is sent to a user, we first check if the user has opted in to receive this type of notification. This information is stored in the notification setting table, with the following fields:

<figure><img src=".gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

#### Rate limiting

Limit the number of notifications to avoid overwhelming users.

#### Retry mechanism

When a third-party service fails to send a notification, the notification will be added to the message queue for retrying. If the problem persists, an alert will be sent out to developers.

#### Security in push notifications

For iOS or Android apps, appKey and appSecret are used to secure push notification APIs. Only authenticated or verified clients are allowed to send push notifications using our APIs.

#### Monitor queued notifications

A key metric to monitor is the total number of queued notifications. If the number is large, more workers are needed to avoid delay in the notification delivery

#### Events tracking

Notification metrics, such as open rate, click rate, and engagement are important in understanding customer behaviors. Analytics service implements events tracking. Integration between the notification system and the analytics service is usually required

<figure><img src=".gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

### Updated design

<figure><img src=".gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

In this design, many new components are added in comparison with the previous design.

* The notification servers are equipped with two more critical features: authentication and rate-limiting.
* We also add a retry mechanism to handle notification failures. If the system fails to send notifications, they are put back in the messaging queue and the workers will retry for a predefined number of times.
* Notification templates provide a consistent and efficient notification creation process.
* Finally, monitoring and tracking systems are added for system health checks and future improvements.

Step 4 - Wrap up

Besides the high-level design, we dug deep into more components and optimizations.

* Reliability: We proposed a robust retry mechanism to minimize the failure rate.
* Security: AppKey/appSecret pair is used to ensure only verified clients can send notifications.
* Tracking and monitoring: These are implemented at any stage of a notification flow to capture important stats.
* Respect user settings: Users may opt-out of receiving notifications. Our system checks user settings first before sending notifications.
* Rate limiting: Users will appreciate a frequency capping on the number of notifications they receive.
