I kind liked it the idea of being writing about socials, but at this time I went to a total different direction by reaching `Twitch.tv` or any type of platform with instant messaging. 

If you’re just starting working with databases, you might want to start off by reading my initial post, [Database 101: Data Consistency for Beginners](https://dev.to/danielhe4rt/database-101-why-so-interesting-1344). That captures my own exploration of how many database paradigms exist as I look far beyond my previous experience with just SQL and MySQL. I’m keeping track on my studies in this **Database 101** series.

## Table Of Contents

- [1. Prologue](#1-prologue)
- [2. The Project: Twitch Sentinel](#2-the-project-twitch-sentinel)
- [3. Making Good Decisions](#3-making-good-decisions)
	- [3.1 ACID Acronym](#31-acid-acronym)
	- [3.2 BASE Acronym](#32-base-acronym)
	- [3.3 Concluding a Good Decision](#33-concluding-a-good-decision)
- [4. Modeling our Ideas into Queries](#4-modeling-our-ideas-into-queries)
	- [4.1 Number of messages per user (top 5)](#41-number-of-messages-per-user-top-5)
	- [4.2 Number of unique users per stream(er)](#42-number-of-unique-users-per-streamer)
	- [4.3 Earliest and latest message of a given user](#43-earliest-and-latest-message-of-a-given-user)
- [5. Twitch as our payload!](#5-twitch-as-our-payload)
- [6. Make it BURN!](#6-make-it-burn)
- [7. Final Considerations](#7-final-considerations)

## 1. Prologue 

One of my tasks as a Developer Advocate is to create and inspire developers to create and try new things. I also need to challenge myself to create interesting things that will engage people.

In the past few weeks, I had a presentation at ScyllaDB called **ScyllaDB from the Developer Perspective: Building Applications**. At first, I didn't understand what my role was in the presentation, since it was my first international presentation. However, once I realized that my role was to build a simple application that would consume 1k/3k operations with the database, the task became easier.

And at the end of your reading, you, a **Beginner Developer** (or maybe not, let me know on the comments) can build an application with a cool database to consume tons of IO/s. 

## 2. The Project: Twitch Sentinel

One of the things that I really like to do in my free time are:
* Create useless PoCs (Proof of Concepts) that help me understand more about concepts in **computer science**.
* Do daily live coding at Twitch.tv ([here is my channel](https://twitch.tv/danielhe4rt)).
* Write blog posts about my learnings and experiences.

Thinking on that, I decided to start a Twitch project that involves a high data ingestion called `Twitch Sentinel`. 

The idea is simple: collect **all the messages** from **many channels** as possible on Twitch.tv, store it and retrieve some metrics from it.

<p align="center" style="font-style: italic;">
	<img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nuvib8bcltl32xv9xnye.png">
	Screenshot of my Twitch chat saying "hi" to you. lol
</p>

Can you imagine any way to store more than 1k messages per second? It sounds like a cool challenge. Here are a few metrics that you can generate for your study:

- How many messages a specific streamer receives on their chat daily.
- How many messages an user sends per day.
- The most probable hour that a streamer or user is online, based on their messages.
- Peak of messages by hour, and so on.

But you might be asking, "Why should I create that if Twitch gives me the Analytics Panel?" The answer is: **USELESS PoCs that will teach you something new!**

Have you ever created an analytics dashboard before? With a really fast database? Thinking of new ways to model it? If the answer is **YES**, drop a like on this post! And if not, let me teach you what I've learned about this subject!

## 3. Making Good Decisions

When you start a new project, you should know that every part of the code and the tools that you will use to deliver it, will impact directly in the future of your project.

Also, the database that you will choose for it implies directly on the performance of the entire application.

Probably if you read my [first post]() of the series **Database 101** you saw something about **CAP Theorem**, **Database Paradigms** and now we're about to check some **properties** that relates on how fast or consistent a database can be.

### 3.1 ACID Acronym

If you're interested in Data Modeling, you probably already heard about `ACID`. If not, `ACID` is an acronym to `Atomicity`, `Consistency`, `Isolation` and `Durability`. Each item of this acronym make up something called `Transaction` in Relational Databases.

A `Transaction`  a single operation that tells your Database that if any of these concepts of ACID fails, your query will be not succeeded. But let's understand why and all the meanings: 

* **Atomicity:** Each piece of your transaction is unique and if any of these pieces fails, it will fail and rollback to the original state of that piece of data.
* **Consistency:** Guarantee that your data will only be accepted if in the right shape of that model, based on: table modeling, triggers, constraints etc.
* **Isolation:** All transactions are independent and will not interfere in a running transaction.
* **Durability:** If the transaction is finished, the data will be persisted even if the system fails after that.

As you can see, ACID gives you an interest safety guard for all your data, but this also have some tradeoffs:

- It's **EXPENSIVE** to maintain, since for each transaction you will LOCK (Isolation) your data. It will need many nodes
- If your goal is to speed things up with higher throughput, ACID will kept you from since all these operations needs to be succeeded.
- You can't even think in **Flexible Schemas**, a.k.a "Documents" since it broke the Consistency property.

So what? We just start to write JSON or TXT as our database? Well, it's not a bad thing but let's do with **Non Relational Databases** (a.k.a **NoSQL**.).

### 3.2 BASE Acronym

When we jump into NoSQL Databases, probably the most important thing to know is: there's paradigms and each paradigm is strong for something specific.

But something that is common between most NoSQL Database Paradigms is the **BASE** properties, which the acronym stands for: **Basically Available**, **Soft State** and **Eventually Consistency**. 

Now things get interesting, because those properties allows you to do "less validation" in your data, since you guarantee the integrity by the developer side. But before talk about that,  let's understand why and all the meanings: 

* **Basically Available:** You can Read/Write data from your database anytime you want, but it doesn't mean that this data is updated.
* **Soft State:** You can literally shape your schema on the fly mostly and it turns into a Developer task, not a DB Administrator anymore.
* **Eventually Consistency:** All the data will be synced between many datacenters until it gets the **Consistency Level** needed.

As it seems, **BASE** properties gives us a database that can receive any type of data regarding the data modeling. Also it's always available to query it and it's a good option for highly data ingestions. But there also tradeoffs:

* If you need strong data consistency, may be not the best approach;
* Higher complexity since the DB Modeling is delegated to the Developer;
* Data Conflicts can be a pain if you need to sync the nodes around the clusters that you have.

Now we know more about a few properties that a Database can have generally have, so it's time to decide based on our project.

### 3.3 Concluding a Good Decision

**ACID** vs **BASE**? Who wins? The type of project decides! In most part of the time, you can use more than one database inside a project and that's not a problem at all. But if you need only one, choose wisely. So just to have it clear: 

* **ACID** should be chosen when you NEED TO HAVE the Data Consistency with Transactions and don't need exactly to be fast.
* **BASE** should be chosen when you have a higher demand for Inputs/Outputs and have sure about your Data Modeling.

On our project, we're gonna receive from `Twitch.tv` an huge amount of messages/s and we need to store it quickly to handle all these messages, of course we're gonna abandon the `ACID 'Safety Guard'` and jump into `BASE 'Do Whatever Wisely'` xD

Thinking on that, I decided to use **CQL** and **ScyllaDB** since it handle our idea of receiving millions of messages per second and at the same time have the **Consistency** and support to **ACID**. If [Discord uses ScyllaDB it for storing messages](https://discord.com/blog/how-discord-stores-trillions-of-messages), why not use it with Twitch? :D

## 4. Modeling our Ideas into Queries

<p align="center" style="font-style: italic;">
	<img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/brs6zzjlcf8grhi62tnr.png">
	Screenshot of my Twitch chat saying "hi" to you. lol
</p>



When you're using ScyllaDB, your main focus needs to be on which query you want to run. Thinking on that, we need to:
- Store messages and read it when necessary.
- Store and read often a streamer list.
- Count all messages sent by chat users in a specific stream;

So our data modeling should be like: 
![aaa](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q787zl9tiie24fe77weq.png)

No big deal here, it's supposed to be simple. We need the fastest throughput possible and complex queries doesn't allow us to have such a performance.

Here is a few queries that we can run with the data retrieved from this model:

### 4.1 Number of messages per user (top 5)

```sql
SELECT
	chatter_id,
	chatter_username,
	count(*) AS msg_count
FROM
	dev_sentinel.messages
GROUP BY
	chatter_id,
	chatter_username
ORDER BY
	msg_count DESC
LIMIT 5
```

### 4.2 Number of unique users per stream(er)
```sql
SELECT
	streamer_id,
	COUNT(DISTINCT chatter_id) AS unique_chatters
FROM
	dev_sentinel.messages
GROUP BY
	streamer_id
ORDER BY
	unique_chatters DESC
```

This is very similar at the previous post modeling and there I explain better a few pieces of the code. Feel free to take a look :p

### 4.3 Earliest and latest message of a given user

SELECT
	min(sent_at) AS earliest_msg,
	max(sent_at) AS latest_msg
FROM
	dev_sentinel.messages
WHERE
	chatter_username = 'danielhe4rt'

## 5. Twitch as our payload!

Ok, now we have the Database concept modeled and now we need the "real world" payload. I don't know if you like these tutorials that just **mock all the data** just to show you a huge number that doesn't even mean nothing at the end... I simply don't and that's why I wanted to bring something real for you to explore.

On Twitch, streaming platform, they have bunch of API's that can interact with the streamer chat. The most notorious is called **'TMI'** - Twitch Messaging Interface - which is a client that connects directly on any Twitch Streamer chat that you want. Here's a list of clients for you check it out:

- [tmi.js](https://tmijs.com/) - Interface for NodeJS
- [tmi.php](tmiphp.com/) - Interface for PHP (Easily integrated to Laravel)
- [pytmi](https://pypi.org/project/pytmi/) - Interface for Python
- [twitch-irc-rs](https://docs.rs/twitch-irc/latest/twitch_irc/) Interface for Rust

Anyway, the idea is the same for all of these clients: you need to choose a channel and connect to it. The code looks like this:

```php

$client = new Client(new ClientOptions([
    'connection' => [
        'secure' => true,
        'reconnect' => true,
        'rejoin' => true,
    ],
    'channels' => ['danielhe4rt']
]));

$client->on(MessageEvent::class, function (MessageEvent $e) {
    print "{$e->tags['display-name']}: {$e->message}";
});

$client->connect();

```

Each Twitch payload has the "tags" array, that brings us a JSON with all the data related to that specific message: 

```json
{
    "badge-info": {
        "subscriber": "58"
    },
    "badges": {
        "broadcaster": "1",
        "subscriber": "3036",
        "partner": "1"
    },
    "client-nonce": "3e00905ed814fb4d846e8b9ba6a9c1da",
    "color": "#8A2BE2",
    "display-name": "danielhe4rt",
    "emotes": null,
    "first-msg": false,
    "flags": null,
    "id": "b40513ae-efed-472b-9863-db34cf0baa98",
    "mod": false,
    "returning-chatter": false,
    "room-id": "227168488",
    "subscriber": true,
    "tmi-sent-ts": "1686770892358",
    "turbo": false,
    "user-id": "227168488",
    "user-type": null,
    "emotes-raw": null,
    "badge-info-raw": "subscriber/58",
    "badges-raw": "broadcaster/1,subscriber/3036,partner/1",
    "username": "danielhe4rt",
    "message-type": "chat"
}

```

On this payload, we gonna need only:
- **room-id**: Identifier related to the specific broadcaster channel.
- **user-id**: Identifier related to the user who sent the message
- **tmi-sent-at**: message timestamp

On the Message interface, you will also receive the string with the message

This is a simple project, but seriously, try to abstract more ideas from that and let me know! I'll gladly help you to create something bigger!


## 6. Make it BURN!

As I told at the beginning of this article, my goal with this project was to build a high scalable application with a really cool database that handles our needs by receiving a huge chunk of payload per second.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/of5jtvh63dxxu4z2gilq.png)

So, we connected on the 20k~ most accessed chats on Twitch.tv and got an average of 1.700 ~ 2.000 messages per second. This gives us an average of 6 MILLION messages per hour. Do you ever coded something that had such a highly data ingestion like that? 


While the application is receiving all these data and posting it to ScyllaDB, here is some statistics of a **T3-Micro Cluster**, the most cheap instance at AWS.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hrvh8y8q55jxnu4q88rx.png)

It handles 1k requests/s like it's nothing having the latency below 1ms (called P99). Also the load of this lightweight machine for 1k/s is just 8%, so you can do something monstrously faster if you want.

The most part of the time, it will depends on how many streamers will be connected on your bot and how many messages viewers send per second.

## 7. Final Considerations

This project teach me a lot about how important is to choose the right tool for the right job. In this specific case, the database needs to be something that you will use thinking in a higher scale. 

Just remember that is totally fine to have more than one database inside a project. Each one resolves a generic or a specific problem inside the development environment. Always do a proper research and PoC with many tools as possible if you have time!

If you want to build by yourself, check this tutorial at Killercoda and don't forget to follow me at socials!


