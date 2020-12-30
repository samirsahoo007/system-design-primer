# URL shortener System design(bit.ly or tinyurl.com)

URL shortening service, a web service that provides short aliases for redirection of long URLs.
```
Example: Let the original URL be: 
Long URL: https://nlogn.in/horizontal-scaling-and-vertical-scaling/
Short URL is: https://tinyurl.com/tepyk8t

Whenever you will click the second short URL, you will automatically be redirected to the page referred by the long URL. 
```

## Design Goals 
The URL shortening service should have the following features:

### Mandatory Features
* Given a long URL it should be able to generate Unique Short URL.
* Given a short URL it should redirect to the original URL.
* The URL should expire after a timespan.

### Optional Features
* The service should be REST API accessible.
* It should provide analytics features i.e, how many times the URL is visited.
* A user should be able to pick a custom URL.

## URL length :
Shortened URL can be combinations of numbers(0–9) and characters(a-Z) of length 7.

## Data capacity modeling :
The important thing to note is that the number of reading requests will be 1000 times more than the number of write requests.

Suppose, we are going to receive 1B(1 Billion) new URL's shorting requests per month, with a 100:1 read/write ratio. So, the total number of redirections: 100  * 1B = 100B redirections

If we are going to store, each URL(long and corresponding short URL) for almost 10 years, then the numbers of URLs we are going to store are: 1B * 120 = 120B URLs

Now if on average every URL is going to be of 200 bytes, then total storage required: 200B * 120byte = 24TB storages

1. Considering average long URL size of 2KB ie for 2048 characters
2. short URL size of about 17 Bytes for 17 character
3. created_at — 10 bytes
4. expiration_length_in_minutes — 10 bytes
5. which gives a total of 2.037 K Bytes

So have 30 M active users so total size = 30000000 * 2.031 = 60780000 KB = 60.78 GB per month
In a Year 0.7284 TB and in 5 year 3.642 TB of data
Think about the number of reads and writes happens to the system!!!

## Algorithm REST Endpoints
Let's starts by making two functions accessible through REST API:

1. create_shortURL( long_url, api_key, custom_url, expiry_date)
*	long_url: A long URL that needs to be shortened.
*	api_key: A unique API key provided to each user, to protect from the spammers, access and resource control for user etc.
*	custom_url(optional): The custom short link URL, user want to use.
*	expiry_date(optional): The date after which the short URL becomes invalid.
*	Return Value: The short Url generated, or error code in case of the inappropriate parameter.
2. redirect_shortURL( short_url)
*	short_url: The short URL generated from the above function.
*	Return Value: The original long URL, or invalid URL error code.

## Shortening logic :
Given a long URL, how can we find hash function F that maps URL to a short alias

## How to convert?
Let's use base62encoder? Base62 encoder contains A-Z , a-z, 0–9 total( 26 + 26 + 10 = 62) basically converting a base10 number to hash, whenever we get a long URL -> get a random number and convert to base62 and use the hash as short URL id.
or You can use the first few characters of MD5(long_url) hash output.

## Database?
We can use RDBMS which provides ACID properties but the problem is scaling (yes you can shard but that increases the complexity of design) 
Since we anticipate storing billions of rows, and we don’t need to use relationships between objects – a NoSQL store like DynamoDB, Cassandra or Riak is a better choice. A NoSQL choice would also be easier to scale. 

Let's try NoSQL

NOSQL: As you know the data is eventually consistent, but it is easy to scale and it is a highly available database

### Database Design & Choice
Let's see the data we need to store:

**Data Related to user**
*User ID:* A unique user id or API key to make user globally distinguishable.
*Name:* The name of the user.
*Email:* The email id of the user
*Password:* Password of the user to facilitate login feature.
*Creation Date:* The date on which the user was registered.

**Data Related to ShortLink**
*Short Url:* 7 character long unique short URL.
*Original Url:* The original long URL.
*UserId:* The unique user id or API key of the user who created the short URL.
*Expiration Date:* The date after which this short URL should become invalid.

#### Data Storage Observation
1. We will be storing Billions of links.
2. The short link redirection should be fast.
3. Our service is going to be read heavily.
4.  The only relation that is going to exist is which user created which URL and that too is going to accessed very less.

We have two different choices of databases: 1) Relational Databases(MySQL) 2) NoSQL Databases( Cassandra).

In general, **Relational Databases are good if we have lots of complex queires involving joins, but they are slow. NoSQL databases are pathetic at handling the relationship queries but they are faster.**

Now, we don't really need lots of relationship among data, but we do need the fast read and write speed. Hence we will choose NoSQL Database. The key for each row can be the shorturl, because it is going to be globally unique.

##### Database Sharding
To scale out our database, we need to partition it into several machines or nodes, so that it can store information about billions of URLs. Hence now we can store more data in memory because of more machines or nodes. For database sharding we will use hash based partition technique. 

In this technique, we will find the hash of the shorturl we are going to store, and determine the machine/shard in which we are going to store this particular URL. The hash function will randomly distribute the URLs into different partation or shard. We can decide the number of shards we are going to make and then we can choose appropriate hash function that random number representing the partation/shard number.( Ex if we have chosen 512 shards, then hash function will generate a number between [1-512] or [0-511] ).

### Speeding Up the Read Operation
We know that our database is going to be read heavily. Till now we have find the way to speed up the writing process, but the reads are still slow. So we have to find some way to speed up the reading process.

Caching is the solution. We can cache the URLs that are going to be accessed frequently. For example, a url that appears on the trending page of any social networking website. Hence many people are going to visit the url. We can use the caching servies like **memcached**.

Things we need to consider after adding the caching layer:

When the cache is full, we need to replace the URLs in cache with the treanding ones. For this we will use **LRU**(Least Recently Used) policy. The URL in the cace which have been refered the least number of times will be removed.
Synchronizing the cache with the original URL. If the user updates or deletes the original link, the corresponding changes has to be reflected in the cache too.
We can shared the cache too. This will help us store more data in memory because of the more machines. For deciding which thing go to which shard can be done using "Hashing" or "Consistant Hashing".

Now this is now we have speed up our read and write request, but still our system is prone to network bandwidth bottleneck and single point of failure.


### Techniques to store Tiny URL?
**Technique 1**

*	Checks whether generated TinyURL is present in DB (doing get(tiny) on DB).
*	If tinyURL isn't present in DB then put longURL and tinyURL in DB (put(tinyURL, longURL)).
*	Problem: This technique creates race condition because it’s possible that two threads may be simultaneously adding same data to DB and may end up corrupting data.
*	If it was simply one server service then it might work, but we are talking about scale!!

So what's the solution? Adding TinyURL and long URL in DB, if there is no key whose value is equal to TinyURL i.e.

*	putIfAbsent(TinyURL, long URL) or INSERT-IF-NOT-EXIST and it Requires support from DB support
*	In this case, we might can’t use NoSQL as putIfAbsent feature support might not be available as the data is eventual consistency, even DB cant perform this operation

**Technique 2 (MD5 Approach)**

* 	Calculate MD5 of long URL
*	Take the first 7 chars and use that to generate TinyURL
*	Now check the DB (using technique 2 discussed above) to verify that TinyURL is not used already
*	Advantages of MD5 approach: For two users using same long URL,
*	In a random generation, we generate two random TinyURL, so 2 rows in DBIn MD5 technique, MD5 of long URL will be same for both the users and hence same first 43 bits of URL will be same and hence deduping will save some space as we will only create one row, saving space

Again we cant use NOSQL as there is no support for putIfAbsent

**Technique 3 — Counters**

*Single server approach*

*	A single server is responsible for maintaining a counter e.g. database,
*	When the worker host receives request they talk to counter, which returns a unique number and increments the counter.
*	Every worker host gets a unique number which is used to generate TinyURL.

Challenges are the Single point of failure and Single point of bottleneck

Before learning about the next approach lets learn something about Zookeeper

**ZooKeeper** is a distributed coordination service to manage a large set of hosts. Co-ordinating and managing a service in a distributed environment is a complicated process. ZooKeeper solves this issue with its simple architecture and API. ZooKeeper allows developers to focus on core application logic without worrying about the distributed nature of the application.

A distributed application can run on multiple systems in a network at a given time (simultaneously) by coordinating among themselves to complete a particular task in a fast and efficient manner. more workers more work done faster.

**The challenges of using a distributed system to get work done are**

*	Race condition − Two or more machines trying to perform a particular task, and leads to race around condition
*	Deadlock — two or more operations waiting for each other to complete indefinitely
*	Inconsistency − Partial failure of data.

Now let’s talk about how to maintain a counter(which we discussed earlier) when we have distributed hosts using Zookeeper.

*	We take 1st billion combinations from 3.5 trillion combination
*	We have divided the 1st billion into 1000 ranges of 1 million each which is maintained by Zookeeper
*	1 -> 1,000,000 (range 1) 1,000,000 -> 2,000,000 (range 1) .... (range n)999,000,000 -> 1,000,000,000 ((range 1000))
*	Worker thread will come to Zookeeper and ask for an unused range. suppose W1 is assigned range 1, W1 will keep incrementing the counter and generate the unique number and generate TinyURL based on that.
*	When a worker thread exhausts their range they will come to Zookeeper and take a fresh range.
*	Guaranteed No Collision of tiny URL
*	Addition of new worker threads is easy
*	Worse case is we will lose a million combinations (if a worker dies) but that’s not as severe as we have 3.5 trillion combinations.
*	When 1st billion is exhausted, we will take 2nd billion and generate ranges to work with.

Ref: https://www.educative.io/courses/grokking-the-system-design-interview/m2ygV4E81AR

# Design Pastebin.com (or Bit.ly)

*Note: This document links directly to relevant areas found in the [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) to avoid duplication.  Refer to the linked content for general talking points, tradeoffs, and alternatives.*

**Design Bit.ly** - is a similar question, except pastebin requires storing the paste contents instead of the original unshortened url.

## Step 1: Outline use cases and constraints

> Gather requirements and scope the problem.
> Ask questions to clarify use cases and constraints.
> Discuss assumptions.

Without an interviewer to address clarifying questions, we'll define some use cases and constraints.

### Use cases

#### We'll scope the problem to handle only the following use cases

* **User** enters a block of text and gets a randomly generated link
    * Expiration
        * Default setting does not expire
        * Can optionally set a timed expiration
* **User** enters a paste's url and views the contents
* **User** is anonymous
* **Service** tracks analytics of pages
    * Monthly visit stats
* **Service** deletes expired pastes
* **Service** has high availability

#### Out of scope

* **User** registers for an account
    * **User** verifies email
* **User** logs into a registered account
    * **User** edits the document
* **User** can set visibility
* **User** can set the shortlink

### Constraints and assumptions

#### State assumptions

* Traffic is not evenly distributed
* Following a short link should be fast
* Pastes are text only
* Page view analytics do not need to be realtime
* 10 million users
* 10 million paste writes per month
* 100 million paste reads per month
* 10:1 read to write ratio

#### Calculate usage

**Clarify with your interviewer if you should run back-of-the-envelope usage calculations.**

* Size per paste
    * 1 KB content per paste
    * `shortlink` - 7 bytes
    * `expiration_length_in_minutes` - 4 bytes
    * `created_at` - 5 bytes
    * `paste_path` - 255 bytes
    * total = ~1.27 KB
* 12.7 GB of new paste content per month
    * 1.27 KB per paste * 10 million pastes per month
    * ~450 GB of new paste content in 3 years
    * 360 million shortlinks in 3 years
    * Assume most are new pastes instead of updates to existing ones
* 4 paste writes per second on average
* 40 read requests per second on average

Handy conversion guide:

* 2.5 million seconds per month
* 1 request per second = 2.5 million requests per month
* 40 requests per second = 100 million requests per month
* 400 requests per second = 1 billion requests per month

## Step 2: Create a high level design

> Outline a high level design with all important components.

![Imgur](http://i.imgur.com/BKsBnmG.png)

## Step 3: Design core components

> Dive into details for each core component.

### Use case: User enters a block of text and gets a randomly generated link

We could use a [relational database](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms) as a large hash table, mapping the generated url to a file server and path containing the paste file.

Instead of managing a file server, we could use a managed **Object Store** such as Amazon S3 or a [NoSQL document store](https://github.com/donnemartin/system-design-primer#document-store).

An alternative to a relational database acting as a large hash table, we could use a [NoSQL key-value store](https://github.com/donnemartin/system-design-primer#key-value-store).  We should discuss the [tradeoffs between choosing SQL or NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql).  The following discussion uses the relational database approach.

* The **Client** sends a create paste request to the **Web Server**, running as a [reverse proxy](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** server does the following:
    * Generates a unique url
        * Checks if the url is unique by looking at the **SQL Database** for a duplicate
        * If the url is not unique, it generates another url
        * If we supported a custom url, we could use the user-supplied (also check for a duplicate)
    * Saves to the **SQL Database** `pastes` table
    * Saves the paste data to the **Object Store**
    * Returns the url

**Clarify with your interviewer how much code you are expected to write**.

The `pastes` table could have the following structure:

```
shortlink char(7) NOT NULL
expiration_length_in_minutes int NOT NULL
created_at datetime NOT NULL
paste_path varchar(255) NOT NULL
PRIMARY KEY(shortlink)
```

We'll create an [index](https://github.com/donnemartin/system-design-primer#use-good-indices) on `shortlink ` and `created_at` to speed up lookups (log-time instead of scanning the entire table) and to keep the data in memory.  Reading 1 MB sequentially from memory takes about 250 microseconds, while reading from SSD takes 4x and from disk takes 80x longer.<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>

To generate the unique url, we could:

* Take the [**MD5**](https://en.wikipedia.org/wiki/MD5) hash of the user's ip_address + timestamp
    * MD5 is a widely used hashing function that produces a 128-bit hash value
    * MD5 is uniformly distributed
    * Alternatively, we could also take the MD5 hash of randomly-generated data
* [**Base 62**](https://www.kerstner.at/2012/07/shortening-strings-using-base-62-encoding/) encode the MD5 hash
    * Base 62 encodes to `[a-zA-Z0-9]` which works well for urls, eliminating the need for escaping special characters
    * There is only one hash result for the original input and Base 62 is deterministic (no randomness involved)
    * Base 64 is another popular encoding but provides issues for urls because of the additional `+` and `/` characters
    * The following [Base 62 pseudocode](http://stackoverflow.com/questions/742013/how-to-code-a-url-shortener) runs in O(k) time where k is the number of digits = 7:

```
def base_encode(num, base=62):
    digits = []
    while num > 0
      remainder = modulo(num, base)
      digits.push(remainder)
      num = divide(num, base)
    digits = digits.reverse
```

* Take the first 7 characters of the output, which results in 62^7 possible values and should be sufficient to handle our constraint of 360 million shortlinks in 3 years:

```
url = base_encode(md5(ip_address+timestamp))[:URL_LENGTH]
```

We'll use a public [**REST API**](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest):

```
$ curl -X POST --data '{ "expiration_length_in_minutes": "60", \
    "paste_contents": "Hello World!" }' https://pastebin.com/api/v1/paste
```

Response:

```
{
    "shortlink": "foobar"
}
```

For internal communications, we could use [Remote Procedure Calls](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc).

### Use case: User enters a paste's url and views the contents

* The **Client** sends a get paste request to the **Web Server**
* The **Web Server** forwards the request to the **Read API** server
* The **Read API** server does the following:
    * Checks the **SQL Database** for the generated url
        * If the url is in the **SQL Database**, fetch the paste contents from the **Object Store**
        * Else, return an error message for the user

REST API:

```
$ curl https://pastebin.com/api/v1/paste?shortlink=foobar
```

Response:

```
{
    "paste_contents": "Hello World"
    "created_at": "YYYY-MM-DD HH:MM:SS"
    "expiration_length_in_minutes": "60"
}
```

### Use case: Service tracks analytics of pages

Since realtime analytics are not a requirement, we could simply **MapReduce** the **Web Server** logs to generate hit counts.

**Clarify with your interviewer how much code you are expected to write**.

```
class HitCounts(MRJob):

    def extract_url(self, line):
        """Extract the generated url from the log line."""
        ...

    def extract_year_month(self, line):
        """Return the year and month portions of the timestamp."""
        ...

    def mapper(self, _, line):
        """Parse each log line, extract and transform relevant lines.

        Emit key value pairs of the form:

        (2016-01, url0), 1
        (2016-01, url0), 1
        (2016-01, url1), 1
        """
        url = self.extract_url(line)
        period = self.extract_year_month(line)
        yield (period, url), 1

    def reducer(self, key, values):
        """Sum values for each key.

        (2016-01, url0), 2
        (2016-01, url1), 1
        """
        yield key, sum(values)
```

### Use case: Service deletes expired pastes

To delete expired pastes, we could just scan the **SQL Database** for all entries whose expiration timestamp are older than the current timestamp.  All expired entries would then be deleted (or  marked as expired) from the table.

## Step 4: Scale the design

> Identify and address bottlenecks, given the constraints.

![Imgur](http://i.imgur.com/4edXG0T.png)

**Important: Do not simply jump right into the final design from the initial design!**

State you would do this iteratively: 1) **Benchmark/Load Test**, 2) **Profile** for bottlenecks 3) address bottlenecks while evaluating alternatives and trade-offs, and 4) repeat.  See [Design a system that scales to millions of users on AWS](../scaling_aws/README.md) as a sample on how to iteratively scale the initial design.

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

The **Analytics Database** could use a data warehousing solution such as Amazon Redshift or Google BigQuery.

An **Object Store** such as Amazon S3 can comfortably handle the constraint of 12.7 GB of new content per month.

To address the 40 *average* read requests per second (higher at peak), traffic for popular content should be handled by the **Memory Cache** instead of the database.  The **Memory Cache** is also useful for handling the unevenly distributed traffic and traffic spikes.  The **SQL Read Replicas** should be able to handle the cache misses, as long as the replicas are not bogged down with replicating writes.

4 *average* paste writes per second (with higher at peak) should be do-able for a single **SQL Write Master-Slave**.  Otherwise, we'll need to employ additional SQL scaling patterns:

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
