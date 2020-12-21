# Design the Twitter timeline and search

*Note: This document links directly to relevant areas found in the [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) to avoid duplication.  Refer to the linked content for general talking points, tradeoffs, and alternatives.*

**Design the Facebook feed** and **Design Facebook search** are similar questions.

# 1. Discuss About The Core Features
So firstly divide the whole system into several core components and talk about some core features. If some other features your interviewer wants to include he/she will mention there. For now, we are going to consider the following features on Twitter…

* The user should be able to tweet in just a few seconds.
* The user should be able to see Tweet Timeline(s)
* **Timeline**: This can be divided into three parts…
	1. User timeline: User see his/her own tweets and tweets user retweet. Tweets which user see when they visit on their profile.
	2. Home timeline: This will display the tweets from people user follow. (Tweets when you land on twitter.com)
	3. Search timeline: When user serch some keywords or #tags and they see the tweets related to that particluar keywords.
* The user should be able to follow another user.
* Users should be able to tweet millions of followers within a few seconds (5 seconds)

# 2. Naive Solution (Synchronous DB queries)
To design a big system like Twitter we will firstly talk about the Naive solution. That will help us in moving towards high-level architecture. You can design a solution for the two things:

	1. **Data modeling:** You can use a relational database like MySQL and you can consider two tables user table (id, username) and a tweet table[id, content, user(primary key of of user table)]. User information will be stored in the user table and whenever a user will tweet a message it will be store in the tweet table. Two relations are also necessary here. One is the user can follow each other, the other is each feed has a user owner. So there will be a one-to-many-relationship between user and tweet table.
	2. **Serve feeds:** You need to fetch all the feeds from all the people a user follows and render them in chronological order.

# 3. Limitation of Architecture (Point Out Bottleneck)
You will have to do a big **select** statement in the tweet table to get all the tweets for a specific user whomsoever he/she is following, and that’s also in chronological order. Doing this every time will create a problem because the tweet table will have huge content with lots of tweets. We need to optimize this solution to solve this problem and for that, we will move to the high-level solution for this problem. Before that let’s understand the characteristics of Twitter first.

# 4. Characteristics of Twitter (Traffic)
Twitter has **300M** daily active users. On average, every second around **6,000** tweets are tweeted on Twitter. Every second **6, 00, 000** Queries made to get the timelines. Each user has on average 200 followers and some users like some celebrities have millions of followers. This characteristic of twitter clears the following points…

	1. Twitter has heavy read in comparison to write so we should care much more about the availability and scale of the application for the heavy read on twitter.
	2. We can consider eventual consistency for this kind of system. It’s completely ok if a user sees the tweet of his follower a bit delayed
	3. Space is not a problem as tweets are limited to 140 characters.

Like we have discussed that twitter is read-heavy so we need a system that allows us to read the information faster and also it can scale horizontally. Redis is perfectly suitable for this requirement but we can not solely dependent on Redis because we also need to store a copy of tweet and other users related info in the Database. So here we will have the basic architecture of Twitter that consists of three tables…User Table, Tweet Table, and Followers Table.

* Whenever a user will create a profile on twitter the entry will be stored in the User table.
* Tweets twitted by a user will be stored in the Tweet table along with the User_id. Also, the User table will have 1 to many relationships with the Tweet table.
* When a user follows another user, it gets stored in Followers Table, and also cache it Redis. The User table will have 1 to many relationships with the Follower table.

# i. User Timeline Architecture
* To get the User Timeline simply go to the user table get the user_id, match this user_id in the tweet table and then get all the tweets. This will also include retweets, save retweets as tweets with original tweet reference. Once this is done sort the tweet by date and time and then display the information on the user timeline.

* As we have discussed that Twitter is read-heavy so the above approach won’t work always. Here we need to use another layer i.e caching layer and we will save the data for user timeline queries in Redis. Also, keep saving the tweets in Redis, so when someone visits a user timeline he/she can get all the tweets made by that user. Getting the data from Redis is much faster so it’s not much use to get it from DB always.

# ii. Home Timeline Architecture
* A user Home Timeline contains all the latest tweets of the person and the pages that the user follows. Well, here you can simply fetch the users whom a user is following, for each follower fetch all the latest tweets, then merge all the tweets, sort all these tweets by date and time and display it on the home timeline. This solution has some drawbacks. The Twitter home page loads much faster and these queries are heavier on the database so this huge search operation will take much more time once the tweet table grows to millions. Let’s talk about the solution now for this drawback…

**Fanout Approach:** Fanout simply means spreading the data from a single point. Let’s see how to use it. Whenever a tweet is made by a user (Followee) do a lot of preprocessing and distribute the data into different users (followers) home timelines. In this process, you won’t have to make any database queries. You just need to go to the cache by user_id and access the home timeline data in Redis. So this process will be much faster and easier because it is an in-memory we get the list of tweets. Here is the complete flow of this approach…
	1. User X is followed by three people and this user has a cache called user timeline. X Tweeted something.
	2. Through Load Balancer tweet will flow into back-end servers.
	3. Server node will save tweet in DB/cache
	4. Server node will fetch all the users that follow User X from the cache.
	5. Server node will inject this tweet into in-memory timelines of his followers (fanout)
	6. All followers of User X will see the tweet of User X in their timeline. It will be refreshed and updated everytime a user will visit on his/her timeline.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/Fanout-System-Design-Twitter.png)

### What will happen if a celebrity will have millions of followers? Is the above method efficient in this scenario?
**Weakness (Edge Case):** The interviewer may ask the above question. If there is a celebrity who has millions of followers then Twitter can take up to 3-44 minutes for a tweet to flow from Eminem(a celebrity) to his million followers. You will have to update the millions of home timelines of followers which is not scalable. Here is the solution…
Solution [Mixed Approach (In-memory+Synchronous calls)]:

	1. Precompute the home timeline of User A (follower of Eminem) with everyone except Eminem’s tweet(s)
	2. For every user maintains the list of celebrities in the cache as well whom that user is following. When the request will arrive (tweet from the celebrity) you can get the celebrity from the list, fetch the tweet from the user timeline of the celebrity and then mix the celebrity’s tweet at runtime with other tweets of User A.
	3. So when User A access his home timeline his tweet feed is merged with Eminem’s tweet at load time. So the celebrity tweet will be inserted at runtime.

**Other Optimization:** For inactive users don’t compute the timeline. People who don’t log in to the system for a quite long time (say more than 20 days.).

# iii. Searching
Tweeter handles searching for its tweets and #tags using Earlybird which is a real-time, reverse index based on Lucene. Early Bird does an inverted full-text indexing operation. It means whenever a tweet is posted it is treated as a document. The tweet will be split into tags, words, and #tags, and then these words are indexed. This indexing is done at a big table or distributed table. In this table, each word has a reference to all the tweets which contain that particular word. Since the index is an exact string-match, unordered, it can be extremely fast. Suppose if a user searches for ‘election‘ then you will go through the table, you will find the word ‘election‘, then you’ll figure out all the references to all the tweets in the system, then it gives all the results that contain the word ‘election‘.

Twitter handles thousands of tweets per second so you can’t have just one big system or table to handle all the data so it should be handled through a distributed approach. Twitter uses the strategy scatter and gather where it set up the multiple servers or data center which allow indexing. When twitter gets a query (let’s say #geeksforgeeks) it sends the query to all the servers or data centers and it queries every Early Bird shard. All the early bird that matches with the query return the result. The results are returned, sorted, merged, and reranked. The ranking is done based on the number of retweets, replies, and the popularity of the tweets.

So far we have talked about all the core features and components of Twitter. There are some other in-depth components you can talk about. For example, you can talk about Trends / Trending topics (using Apache Storm and Heron framework), you can talk about notifications and how to incorporate advertisement.





## Step 1: Outline use cases and constraints

> Gather requirements and scope the problem.
> Ask questions to clarify use cases and constraints.
> Discuss assumptions.

Without an interviewer to address clarifying questions, we'll define some use cases and constraints.

### Use cases

#### We'll scope the problem to handle only the following use cases

* **User** posts a tweet
    * **Service** pushes tweets to followers, sending push notifications and emails
* **User** views the user timeline (activity from the user)
* **User** views the home timeline (activity from people the user is following)
* **User** searches keywords
* **Service** has high availability

#### Out of scope

* **Service** pushes tweets to the Twitter Firehose and other streams
* **Service** strips out tweets based on user's visibility settings
    * Hide @reply if the user is not also following the person being replied to
    * Respect 'hide retweets' setting
* Analytics

### Constraints and assumptions

#### State assumptions

General

* Traffic is not evenly distributed
* Posting a tweet should be fast
    * Fanning out a tweet to all of your followers should be fast, unless you have millions of followers
* 100 million active users
* 500 million tweets per day or 15 billion tweets per month
    * Each tweet averages a fanout of 10 deliveries
    * 5 billion total tweets delivered on fanout per day
    * 150 billion tweets delivered on fanout per month
* 250 billion read requests per month
* 10 billion searches per month

Timeline

* Viewing the timeline should be fast
* Twitter is more read heavy than write heavy
    * Optimize for fast reads of tweets
* Ingesting tweets is write heavy

Search

* Searching should be fast
* Search is read-heavy

#### Calculate usage

**Clarify with your interviewer if you should run back-of-the-envelope usage calculations.**

* Size per tweet:
    * `tweet_id` - 8 bytes
    * `user_id` - 32 bytes
    * `text` - 140 bytes
    * `media` - 10 KB average
    * Total: ~10 KB
* 150 TB of new tweet content per month
    * 10 KB per tweet * 500 million tweets per day * 30 days per month
    * 5.4 PB of new tweet content in 3 years
* 100 thousand read requests per second
    * 250 billion read requests per month * (400 requests per second / 1 billion requests per month)
* 6,000 tweets per second
    * 15 billion tweets per month * (400 requests per second / 1 billion requests per month)
* 60 thousand tweets delivered on fanout per second
    * 150 billion tweets delivered on fanout per month * (400 requests per second / 1 billion requests per month)
* 4,000 search requests per second

Handy conversion guide:

* 2.5 million seconds per month
* 1 request per second = 2.5 million requests per month
* 40 requests per second = 100 million requests per month
* 400 requests per second = 1 billion requests per month

## Step 2: Create a high level design

> Outline a high level design with all important components.

![Imgur](http://i.imgur.com/48tEA2j.png)

OR

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/High-Level-Solution-for-Twitter-System-Design.png)

## Step 3: Design core components

> Dive into details for each core component.

### Use case: User posts a tweet

We could store the user's own tweets to populate the user timeline (activity from the user) in a [relational database](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms).  We should discuss the [use cases and tradeoffs between choosing SQL or NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql).

Delivering tweets and building the home timeline (activity from people the user is following) is trickier.  Fanning out tweets to all followers (60 thousand tweets delivered on fanout per second) will overload a traditional [relational database](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms).  We'll probably want to choose a data store with fast writes such as a **NoSQL database** or **Memory Cache**.  Reading 1 MB sequentially from memory takes about 250 microseconds, while reading from SSD takes 4x and from disk takes 80x longer.<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>

We could store media such as photos or videos on an **Object Store**.

* The **Client** posts a tweet to the **Web Server**, running as a [reverse proxy](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** stores the tweet in the user's timeline on a **SQL database**
* The **Write API** contacts the **Fan Out Service**, which does the following:
    * Queries the **User Graph Service** to find the user's followers stored in the **Memory Cache**
    * Stores the tweet in the *home timeline of the user's followers* in a **Memory Cache**
        * O(n) operation:  1,000 followers = 1,000 lookups and inserts
    * Stores the tweet in the **Search Index Service** to enable fast searching
    * Stores media in the **Object Store**
    * Uses the **Notification Service** to send out push notifications to followers:
        * Uses a **Queue** (not pictured) to asynchronously send out notifications

**Clarify with your interviewer how much code you are expected to write**.

If our **Memory Cache** is Redis, we could use a native Redis list with the following structure:

```
           tweet n+2                   tweet n+1                   tweet n
| 8 bytes   8 bytes  1 byte | 8 bytes   8 bytes  1 byte | 8 bytes   8 bytes  1 byte |
| tweet_id  user_id  meta   | tweet_id  user_id  meta   | tweet_id  user_id  meta   |
```

The new tweet would be placed in the **Memory Cache**, which populates user's home timeline (activity from people the user is following).

We'll use a public [**REST API**](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest):

```
$ curl -X POST --data '{ "user_id": "123", "auth_token": "ABC123", \
    "status": "hello world!", "media_ids": "ABC987" }' \
    https://twitter.com/api/v1/tweet
```

Response:

```
{
    "created_at": "Wed Sep 05 00:37:15 +0000 2012",
    "status": "hello world!",
    "tweet_id": "987",
    "user_id": "123",
    ...
}
```

For internal communications, we could use [Remote Procedure Calls](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc).

### Use case: User views the home timeline

* The **Client** posts a home timeline request to the **Web Server**
* The **Web Server** forwards the request to the **Read API** server
* The **Read API** server contacts the **Timeline Service**, which does the following:
    * Gets the timeline data stored in the **Memory Cache**, containing tweet ids and user ids - O(1)
    * Queries the **Tweet Info Service** with a [multiget](http://redis.io/commands/mget) to obtain additional info about the tweet ids - O(n)
    * Queries the **User Info Service** with a multiget to obtain additional info about the user ids - O(n)

REST API:

```
$ curl https://twitter.com/api/v1/home_timeline?user_id=123
```

Response:

```
{
    "user_id": "456",
    "tweet_id": "123",
    "status": "foo"
},
{
    "user_id": "789",
    "tweet_id": "456",
    "status": "bar"
},
{
    "user_id": "789",
    "tweet_id": "579",
    "status": "baz"
},
```

### Use case: User views the user timeline

* The **Client** posts a user timeline request to the **Web Server**
* The **Web Server** forwards the request to the **Read API** server
* The **Read API** retrieves the user timeline from the **SQL Database**

The REST API would be similar to the home timeline, except all tweets would come from the user as opposed to the people the user is following.

### Use case: User searches keywords

* The **Client** sends a search request to the **Web Server**
* The **Web Server** forwards the request to the **Search API** server
* The **Search API** contacts the **Search Service**, which does the following:
    * Parses/tokenizes the input query, determining what needs to be searched
        * Removes markup
        * Breaks up the text into terms
        * Fixes typos
        * Normalizes capitalization
        * Converts the query to use boolean operations
    * Queries the **Search Cluster** (ie [Lucene](https://lucene.apache.org/)) for the results:
        * [Scatter gathers](https://github.com/donnemartin/system-design-primer#under-development) each server in the cluster to determine if there are any results for the query
        * Merges, ranks, sorts, and returns the results

REST API:

```
$ curl https://twitter.com/api/v1/search?query=hello+world
```

The response would be similar to that of the home timeline, except for tweets matching the given query.

## Step 4: Scale the design

> Identify and address bottlenecks, given the constraints.

![Imgur](http://i.imgur.com/jrUBAF7.png)

**Important: Do not simply jump right into the final design from the initial design!**

State you would 1) **Benchmark/Load Test**, 2) **Profile** for bottlenecks 3) address bottlenecks while evaluating alternatives and trade-offs, and 4) repeat.  See [Design a system that scales to millions of users on AWS](../scaling_aws/README.md) as a sample on how to iteratively scale the initial design.

It's important to discuss what bottlenecks you might encounter with the initial design and how you might address each of them.  For example, what issues are addressed by adding a **Load Balancer** with multiple **Web Servers**?  **CDN**?  **Master-Slave Replicas**?  What are the alternatives and **Trade-Offs** for each?

We'll introduce some components to complete the design and to address scalability issues.  Internal load balancers are not shown to reduce clutter.

*To avoid repeating discussions*, refer to the following [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) for main talking points, tradeoffs, and alternatives:

* [DNS](https://github.com/donnemartin/system-design-primer#domain-name-system)
* [CDN](https://github.com/donnemartin/system-design-primer#content-delivery-network)
* [Load balancer](https://github.com/donnemartin/system-design-primer#load-balancer)
* [Horizontal scaling](https://github.com/donnemartin/system-design-primer#horizontal-scaling)
* [Web server (reverse proxy)](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* [API server (application layer)](https://github.com/donnemartin/system-design-primer#application-layer)
* [Cache](https://github.com/donnemartin/system-design-primer#cache)
* [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)
* [SQL write master-slave failover](https://github.com/donnemartin/system-design-primer#fail-over)
* [Master-slave replication](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [Consistency patterns](https://github.com/donnemartin/system-design-primer#consistency-patterns)
* [Availability patterns](https://github.com/donnemartin/system-design-primer#availability-patterns)

The **Fanout Service** is a potential bottleneck.  Twitter users with millions of followers could take several minutes to have their tweets go through the fanout process.  This could lead to race conditions with @replies to the tweet, which we could mitigate by re-ordering the tweets at serve time.

We could also avoid fanning out tweets from highly-followed users.  Instead, we could search to find tweets for high-followed users, merge the search results with the user's home timeline results, then re-order the tweets at serve time.

Additional optimizations include:

* Keep only several hundred tweets for each home timeline in the **Memory Cache**
* Keep only active users' home timeline info in the **Memory Cache**
    * If a user was not previously active in the past 30 days, we could rebuild the timeline from the **SQL Database**
        * Query the **User Graph Service** to determine who the user is following
        * Get the tweets from the **SQL Database** and add them to the **Memory Cache**
* Store only a month of tweets in the **Tweet Info Service**
* Store only active users in the **User Info Service**
* The **Search Cluster** would likely need to keep the tweets in memory to keep latency low

We'll also want to address the bottleneck with the **SQL Database**.

Although the **Memory Cache** should reduce the load on the database, it is unlikely the **SQL Read Replicas** alone would be enough to handle the cache misses.  We'll probably need to employ additional SQL scaling patterns.

The high volume of writes would overwhelm a single **SQL Write Master-Slave**, also pointing to a need for additional scaling techniques.

* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

We should also consider moving some data to a **NoSQL Database**.

## Additional talking points

> Additional topics to dive into, depending on the problem scope and time remaining.

#### NoSQL

* [Key-value store](https://github.com/donnemartin/system-design-primer#key-value-store)
* [Document store](https://github.com/donnemartin/system-design-primer#document-store)
* [Wide column store](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [Graph database](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL vs NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

### Caching

* Where to cache
    * [Client caching](https://github.com/donnemartin/system-design-primer#client-caching)
    * [CDN caching](https://github.com/donnemartin/system-design-primer#cdn-caching)
    * [Web server caching](https://github.com/donnemartin/system-design-primer#web-server-caching)
    * [Database caching](https://github.com/donnemartin/system-design-primer#database-caching)
    * [Application caching](https://github.com/donnemartin/system-design-primer#application-caching)
* What to cache
    * [Caching at the database query level](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
    * [Caching at the object level](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)
* When to update the cache
    * [Cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside)
    * [Write-through](https://github.com/donnemartin/system-design-primer#write-through)
    * [Write-behind (write-back)](https://github.com/donnemartin/system-design-primer#write-behind-write-back)
    * [Refresh ahead](https://github.com/donnemartin/system-design-primer#refresh-ahead)

### Asynchronism and microservices

* [Message queues](https://github.com/donnemartin/system-design-primer#message-queues)
* [Task queues](https://github.com/donnemartin/system-design-primer#task-queues)
* [Back pressure](https://github.com/donnemartin/system-design-primer#back-pressure)
* [Microservices](https://github.com/donnemartin/system-design-primer#microservices)

### Communications

* Discuss tradeoffs:
    * External communication with clients - [HTTP APIs following REST](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
    * Internal communications - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [Service discovery](https://github.com/donnemartin/system-design-primer#service-discovery)

### Security

Refer to the [security section](https://github.com/donnemartin/system-design-primer#security).

### Latency numbers

See [Latency numbers every programmer should know](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know).

### Ongoing

* Continue benchmarking and monitoring your system to address bottlenecks as they come up
* Scaling is an iterative process
