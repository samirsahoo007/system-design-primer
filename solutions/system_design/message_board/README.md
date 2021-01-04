Also refer
Stack Overflow: The Architecture: https://nickcraver.com/blog/2016/02/17/stack-overflow-the-architecture-2016-edition/
https://nickcraver.com/blog/archive/

**Database:**

Posts

Users

Comments

Badges

Votes

Tags

CloseAsOffTopicReasonTypes

PendingFlags

PostFeedback

PostHistory

ReviewTasks

SuggestedEdits

for details refer:
https://meta.stackexchange.com/questions/2677/database-schema-documentation-for-the-public-data-dump-and-sede/2678#2678

# System design: Reddit or Quora or Hacker News

Imagine you're the lead engineer responsible for building Reddit from the ground up. Walk me through how you would design the system to support the following functionality:

## Requirements:

Users can make posts in different forums (i.e. "subreddits")
Users can attach images to their posts
Users can upvote or downvote posts
Users can add comments to posts
Users can see a feed of posts sorted by ranking or recency

## Constraints:

We want to support a large volume of users (millions) viewing and posting content

## Our solution
This is a broad problem with many interesting aspects to explore: hosting user-generated content, voting and ranking, and designing a system at a massive scale. For our solution, we'll follow the approach outlined in the first lesson of this module.

### 1. Define problem space
The first step is to explore and clarify what features and requirements are in scope for the discussion. In this case, we've already been given a list of feature requirements and an idea of the scale that we should support. Even so, you should still dig deeper and try to understand if there are more requirements or use cases. Here are some relevant clarifications:

Do we need to support users on mobile apps or only web?
Will users upload their images to Reddit or link to a third-party image hosting service?
Are there any performance or latency requirements that would impact our design choices?
For now, let's assume that we only care about web users, we do want to host user content directly on our servers, and we want this content to load quickly for users around the world, regardless of location.

### 2. High-level Design

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/reddit_design.svg)

Let's start by picking the core components of our system! We know from the project requirements that we want to allow users to view, post, upvote, and comment. Think about what components of your system you'll need to support this: databases, servers, user interfaces, etc.

Let's start from the database and work our way up. We know we'll need a data store for all of our users, posts, and upvotes, and we'll also need to store and retrieve large image files. For the first type of data, a relational database makes the most sense because there's a clear relational structure—users have many posts, posts have many upvotes, etc. For this reason, it makes sense to pick a SQL database since it is more efficient at modeling and querying for relational data. We can't store arbitrary files in a SQL database, however, so we also need an object storage system, like Amazon S3.

Now that we know how we're going to store our data, we need application servers to perform "CRUD" operations on the underlying data, handle user authentication, and the rest of our business logic. Due to the scale of our system, we'll also need many server instances (or even multiple points of presence), along with a load balancer to distribute traffic across these servers. We'll also need to implement caching layers across the board for common operations like ranking as well as content distribution.

3. Define each part of our system
Next, we'll dive into the details of each component of the system and talk about tradeoffs. It's unlikely that you'll have time in an interview to dive deeply into all of these topics, but you should be prepared to talk at some level about any of them. Remember the interview is a discussion with some give and take.

#### Database Schema
Let's model the data tables we'll need and the relationships among them. For each of these tables, you can list out the main columns we'll need, their type, and any foreign keys or relationship between them.

##### Users

Column  | Type
------------- | -------------
id		|integer
username	|string
email		|string
password_hash	|string
created		|timestamp

##### Posts
| Column | Type  | References |
| :------------ |:---------------:| -----:|
|id		|integer	|	
|user_id	|integer	|user.id
|subreddit_id	|integer	|subreddit.id
|content	|	string	|
|image_url	|string		|
|created	|	timestamp|
|edited		|timestamp	|

##### Subreddits
Column  | Type
------------- | -------------
id		|integer
subreddit_name	|string
description	|string
created		|timestamp

##### Upvotes
| Column | Type  | References |
| :------------ |:---------------:| -----:|
|id		|integer		|
|user_id		|integer	|user.id
|post_id		|integer	|post.id
|created		|timestamp	|

##### Comments
| Column | Type  | References |
| :------------ |:---------------:| -----:|
|id		|integer|
|user_id	|integer|user.id
|post_id	|integer|post.id
|comment_id	|integer|comment.id
|content	|string|
|created	|timestamp|
|edited		|timestamp|

##### Indexing
It's important to mention what fields we'll need to index in our database to support the types of queries we're making. We'll want to add indices on the user_id and subreddit_id columns to support fast lookup of posts. Similarly, we'll want to add indexes on other foreign keys and a unique index to fields such as username and subreddit_name.

##### Sharding
At this scale, we simply cannot read and write all of our data with a single database machine, so we will need to implement a sharding strategy across multiple machines (or use a distributed database service such as Google's Cloud Spanner). Choosing a sharding key is an important decision and highly application-specific. For example, sharding by creation date is probably not suitable in this scenario, since the newest shard will always receive much more traffic. Instead, we may be better off sharding posts by subreddit so that related data can be served and cached from the same machines. We can also use a consistent hashing scheme so that new shards can be easily created.

##### API
Now let's briefly outline the API that our frontend application will use to interact with our servers:

**Endpoint	Method	Options**
/posts	GET	subreddit_name, username, limit, startingAfter
/posts	POST	subreddit_name, post_content
/posts	PATCH	post_id, post_content
/posts	DELETE	post_id
/users	GET	username
/users	POST	username, email, password
/comments	GET	post_id, comment_id
/comments	POST	post_id, comment_id, comment_content
/upvotes	POST	post_id
/upvotes	DELETE	post_id

##### Caching
In order to support the scale and performance requirements of our project, we will need to use multiple types of caching throughout our system:

**Retrieval and ranking**: To improve the performance of retrieval and ranking of posts, we can add a caching layer (like Memcached or Redis) between our application and databases. We can update our cache on a periodic basis, either with a scheduled job or directly upon user-initiated actions like posting and voting. Our choice will depend largely on the estimated volume of views vs. posts and the acceptable latency in ranking updates. To make our cache usage more efficient, we must also consider what "eviction policy" would make sense for our application; one option would be to cache the rankings by subreddit and use a least-recently used (LRU) policy to prioritize more popular subreddits over time.

**Content delivery network**: To deliver static file content, such as user-uploaded images and frontend resources, we will need to make use a distributed content delivery network (CDN). This service will permit us to cache resources at nodes around the world, reducing the load on our backend servers while also decreasing latency for users. Newly uploaded resources can be pushed to the CDN or pulled by the CDN from object storage as needed. Some CDNs also offer additional benefits such as automatic image compression and optimization for different types of devices.

Ref: https://www.tryexponent.com/courses/system-design/design-reddit


# Lessons Learned
This is a mix of lessons taken from Jeff and Joel and comments from their posts.

If you’re comfortable managing servers then buy them. The two biggest problems with renting costs were: 1) the insane cost of memory and disk upgrades 2) the fact that they [hosting providers] really couldn’t manage anything.
Make larger one time up front investments to avoid recurring monthly costs which are more expensive in the long term.
Update all network drivers. Performance went from 2x slower to 2x faster.
Upgrading to 48GB RAM required upgrading MS Enterprise edition.
Memory is incredibly cheap. Max it out for almost free performance. At Dell, for example, upgrading from 4G memory to 128G is $4378.
Stack Overflow copied a key part of the Wikipedia database design. This turned out to be a mistake which will need massive and painful database refactoring to fix. The refactorings will be to avoid excessive joins in a lot of key queries. This is the key lesson from giant multi-terabyte table schemas (like Google’s BigTable) which are completely join-free. This is significant because Stack Overflow's database is almost completely in RAM and the joins still exact too high a cost.
CPU speed is surprisingly important to the database server. Going from 1.86 GHz, to 2.5 GHz, to 3.5 GHz CPUs causes an almost linear improvement in typical query times. The exception is queries which don’t fit in memory.
When renting hardware nobody pays list price for RAM upgrades unless you are on a month-to-month contract.
The bottleneck is the database 90% of the time.
At low server volume, the key cost driver is not rackspace, power, bandwidth, servers, or software; it is NETWORKING EQUIPMENT. You need a gigabit network between your DB and Web tiers. Between the cloud and your web server, you need firewall, routing, and VPN devices. The moment you add a second web server, you also need a load balancing appliance. The upfront cost of these devices can easily be 2x the cost of a handful of servers.
EC2 is for scaling horizontally, that is you can split up your work across many machines (a good idea if you want to be able to scale). It makes even more sense if you need to be able to scale on demand (add and remove machines as load increases / decreases).
Scaling out is only frictionless when you use open source software. Otherwise scaling up means paying less for licenses and a lot more for hardware, while scaling out means paying less for the hardware, and a whole lot more for licenses.
RAID-10 is awesome in a heavy read/write database workload.
Separate application and database duties so each can scale independently of the other. Databases scale up and the applications scale out.
Applications should keep state in the database so they scale horizontally by adding more servers.
The problem with a scale up strategy is a lack of redundancy. A cluster ads more reliability, but is very expensive when the individual machines are expensive.
Few applications can scale linearly with the number of processors. Locks will be taken which serializes processing and ends up reducing the effectiveness of your Big Iron.
With larger form factors like 7U power and cooling become critical issues. Using something between 1U and 7U might be easier to make work in your data center.
As you add more and more database servers the SQL Server license costs can be outrageous. So by starting scale up and gradually going scale out with non-open source software you can be in a world of financial hurt.

It's true there's not much about their architecture here. We know about their machines, their tool chain, and that they use a two-tier architecture where they access the database directly from the web server code. We don't know how they implement tags, etc. If interested you'll be able to glean some of this information from an explanation of their schema.

## NoSQL Is Hard
So should Stack Overflow have scaled out instead of up, just in case?

What some don't realize is NoSQL is hard. Relational databases have many many faults, but they make a lot of common tasks simple while hiding both the cost and complexity. If you want to know how many black Prius cars are in inventory, for example, then that's pretty easy to do.

Not so with most NoSQL databases (I'll speak generally here, some NoSQL databases have more features than others). You would have program a counter of black Prius cars yourself, up front, in code. There are no aggregate operators. You must maintain secondary indexes. There's no searching. There are no distributed queries across partitions. There's no Group By or Order By. There are no cursors for easy paging through result sets. Returning even 100 large records at time may timeout. There may be quotas that are very restrictive because they must limit the amount of IO for any one operation. Query languages may lack expressive power.

The biggest problem of all is that transactions can not span arbitrary boundaries. There are no ACID guarantees beyond a single record or small entity group. Once you wrap your head around what this means for the programmer it's not a pleasant prospect at all. References must be manually maintained. Relationships must be manually maintained. There are no cascading deletes that act correctly during a failure. Every copy of denormalized data must be manually tracked and updated taking into account the possibility of partial failures and externally visible inconsistency.

All this functionality must be written manually by you in your code. While flexibility to write your own code is great in an OLAP/map-reduce situation, declarative approaches still cover a lot of ground and make for much less brittle code.

What you gain is the ability to write huge quantities of data. What you lose is complacency. The programmer must be very aware at all times that they are dealing with a system where it costs a lot to perform distribute operations and failure can occur at anytime.

All this may be the price of building a truly scalable and distributed system, but is this really the price you want to pay?

## The Multitenancy Problem
With StackExchange Stack Overflow has gone into the multi-tenancy business. They are offering StackExchange either self-hosted or as a hosted white label application.

It will be interesting to see if their architecture can scale to handle a large number of sites. Salesorce is the king of multitenancy and although it's true they use Oracle as their database, they basically use very little of Oracle and have written their own table structure, indexing and query processor on top of Oracle. All in order to support multitenancy.

Salesforce went extreme because supporting a lot of different customers is way more difficult than it seems, especially once you allow customization and support versioning.

Clearly all customers can't run in one server for security, customization, and scaling reasons.

You may think just create a database for each customer, share a server for a certain number of customers, and then add more servers as needed. As long as a customer doesn't need more than one server you are golden.

This doesn't seem to work well in practice. Oddly database managers aren't optimized for adding or updating databases. Creating databases is a heavyweight operation and can degrade performance for existing customers as system locks are taken. Upgrade issues are also problematic. Adding columns locks tables which causes problems in high traffic situations. Adding new indexes can also take a very long time and degrade performance. Plus each customer will likely have specializations that makes upgrading even more complicated.

To get around these problems Salesforce's Craig Weissman, Chief Architect, created an innovative approach where tables are not created for each customer. All data from all customers is mapped into the same data table, including indexes. The schema for that table looks something like orgid, oid, value0, value1...value500. "orgid" is the organization ID and is how data is never mixed up. It's a very wide and sparse table, which Oracle seems to handle well. Hundreds and hundreds of "tables" and custom fields are mapped into the data table.

With this approach Salesforce has no option other than to build their own infrastructure to interpret what's in that table. Oracle is left to handle transactions, concurrency, and deadlock detection. The advatange is because there's an interpreted layer handling versions and upgrades is relatively simple because the handling logic can be baked in. Strange but true.
