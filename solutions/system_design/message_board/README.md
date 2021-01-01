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

