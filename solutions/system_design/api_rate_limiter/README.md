
# Design API rate limiter

A rate limiter is a tool that monitors the number of requests per a window time a service agrees to allow. If the request count exceeds the number agreed by the service owner and the user (in a decided window time), the rate limiter blocks all the excess calls(say by throwing exceptions). The user can be a human or any other service(ex: in a micro service based architecture)

**Example:** A device with a specific ip address can only make 10 ticket bookings per minute to a show booking website. Or a service A can hit service B with a rate of at most 500 req/sec. All the excess requests get rejected.

Also Rate limiting protects your APIs from overuse by limiting how often each user can call the API. This protects them from inadvertent or malicious overuse. Without rate limiting, each user may request as often as they like, which can lead to “spikes” of requests that starve other consumers. After rate limiting is enabled, they are limited to a fixed number of requests per second.

![API Rate Limiter](01-rate-limit-kong.png)

In the example chart, you can see how rate limiting blocks requests over time. The API was initially receiving 4 requests per minute shown in green. When rate limiting was enabled at 12:02, additional requests shown in red are denied.

Rate limiting is very important for public APIs where you want to maintain a good quality of service for every consumer, even when some users take more than their fair share. Computationally-intensive endpoints are particularly in need of rate limiting – especially when served by auto-scaling, or by pay-by-the-computation services like AWS Lambda and OpenWhisk. You also may want to rate limit APIs that serve sensitive data, because this could limit the data exposed if an attacker gains access in some unforeseen event.

There are actually many different ways to enable rate limiting, and we will explore the pros and cons of different rate limiting algorithms. We will also explore the issues that come up when scaling across a cluster. Lastly, we’ll show you an example of how to quickly set up rate limiting using Kong, which is the most popular open-source API gateway.

## Problem statement:

Design and write classes need to implement api rate limiter. List out the data structures and design patterns needed for this implementation.

For example rate limiter should allow only 50 reqs/sec for free plan user, 500 reqs/sec for standard plan user and 1000 reqs/sec for pro plan user. Rate limiter should throw error for each REQUEST beyond this rate limit for each SECOND.

Note: This is not complete design and this design is theoritacal and not implemented practically.

Solution:
----------
For these use cases, using persistent memory stores like mysql is a bad idea because the time taken for disk seeks is high enough to hamper the rate limiter granularity. For instance, let’s say that we’re using a single mysql instance to store the request counts and mysql takes 1ms to process 1 request, which means we can achieve a throughput of 1000 req/sec. But a rate limiter using in-memory cache takes around 100nanosec(a main-memory access) to process 1 request, which implies a granularity of around 10Mreq/sec can be achieved.

**Various Levels:** There are different levels at which one can design rate limiters. Listing a few…
* Rate limiter in a single machine, single threaded scenario
* Rate limiter in a single machine, multi threaded scenario — Handling race conditions
* Rate limiter in a distributed scenario —Distributed Cache Usage like redis
* Rate limiter from client side —Prevent network calls from client to server for all the excess requests.

There are different designs to implement rate limiters, some of which are described below…

- The idea is to send all requests through API rate limit handler and processing further. This sounds like using interceptor design pattern.
![API Rate Limiter](api_rate_limiter2.png)
- As shown in above image, all requests will pass through rate limit handler. So for every request we can verify how many requests this user has sent in last one second, minute or hour based on user plan.
- We should use some kind of storage mechanism to count the number of requests/duration. Below is the sample table schema looks like.
![API Requests schema](api_requests_schema.png)
- In the schema, start time and end time are the boundaries for requests. It means let's say a user is in free plan that means from the problem statement we should allow only 50 reqs/sec. So start and end time will be differ by 1000 milli secs.
- We'll have our logic like when API rate limiter receives a request if checks in the database for a record which is least recent in start time having empty end time for the current user. If it found then validates the number of requests and proceeds further if limit is not reached else throws exception.
![API Rate Limiter Flow Chart](api_rate_limiter_flow.png)


# Rate limiter

<!-- MarkdownTOC -->

- [Goals](#goals)
- [Algorithm](#algorithm)
	- [Token bucket](#token-bucket)
	- [Leaky bucket](#leaky-bucket)
	- [Fixed window](#fixed-window)
	- [Sliding log](#sliding-log)
	- [Sliding window](#sliding-window)
- [Single machine rate limit](#single-machine-rate-limit)
	- [Guava rate limiter](#guava-rate-limiter)
		- [Implementation](#implementation)
		- [Producer consumer pattern](#producer-consumer-pattern)
                - [Record the next time a token is available](#record-the-next-time-a-token-is-available)
                        - [Warm up feature](#warm-up-feature)
        - [Ratelimiter within Resiliency4J](#ratelimiter-within-resiliency4j)
- [Distributed rate limit](#distributed-rate-limit)
	- [Sticky sessions](#sticky-sessions)
	- [Nginx based rate limiting](#nginx-based-rate-limiting)
	- [Redis based rate limiter](#redis-based-rate-limiter)
		- [Implementation](#implementation-1)
                        - [Sliding log implementation using ZSet](#sliding-log-implementation-using-zset)
                        - [Sliding window implementation](#sliding-window-implementation)
                        - [Token bucket implementation](#token-bucket-implementation)
                - [Challenges](#challenges)
                        - [How to handle race conditions](#how-to-handle-race-conditions)
                        - [How to handle the additional latency introduce by performance](#how-to-handle-the-additional-latency-introduce-by-performance)
                        - [How to avoid multiple round trips for different buckets:](#how-to-avoid-multiple-round-trips-for-different-buckets)
                        - [Performance bottleneck and single point failure due to Redis](#performance-bottleneck-and-single-point-failure-due-to-redis)
                        - [Static rate limit threshold](#static-rate-limit-threshold)
	- [Ratelimiter within CloudBouncer](#ratelimiter-within-cloudbouncer)
	- [Redis cell rate limiter](#redis-cell-rate-limiter)

<!-- /MarkdownTOC -->


## Goals
* Sharing access to limited resources: Requests made to an API where the limited resources are your server capacity, database load, etc.
* Security: Limiting the number of second factor attempts that a user is allowed to perform, or the number of times they’re allowed to get their password wrong.
* Revenue: Certain services might want to limit actions based on the tier of their customer’s service, and thus create a revenue model based on rate limiting.

## Algorithm
### Token bucket
* The token bucket limits the average inflow rate and allows sudden increase in traffic.
        - Steps
        1. A token is added every t time.
        2. The bucket can hold at most b tokens. If a token arrive when bucket is full the token will be discarded.
        3. When a packet of m bytes arrived m tokens are removed from the bucket and the packet is sent to the network.
        4. If less than n tokens are available no tokens will be removed from the bucket and the packet is considered to be non-comformant.
    - Pros
        - Smooth out the requests and process them at an approximately average rate.
    - Cons
        - A burst of request could fill up the queue with old requests and starve the more recent requests from being processed. Does not guarantee that requests get processed within a fixed amount of time. Consider an antisocial script that can make enough concurrent requests that it can exhaust its rate limit in short order and which is regularly overlimit. Once an hour as the limit resets, the script bombards the server with a new series of requests until its rate is exhausted once again. In this scenario the server always needs enough extra capacity to handle these short intense bursts and which will likely go to waste during the rest of the hour.

### Leaky bucket
* Leaky bucket (closely related to token bucket) is an algorithm that provides a simple, intuitive approach to rate limiting via a queue which you can think of as a bucket holding the requests. When a request is registered, it is appended to the end of the queue. At a regular interval, the first item on the queue is processed. This is also known as a first in first out (FIFO) queue. If the queue is full, then additional requests are discarded (or leaked).
* Steps
    1. Initialize the counter to N at every tick of the clock
    2. If N is greater than the size of the packet in front of the queue send the packet to network and decrement the counter by the size of the packet.
    3. Reset the counter and go to Step - 1.
* Pros:
    - The leaky bucket produces a very smooth rate limiting effect. A user can still exhaust their entire quota by filling their entire bucket nearly instantaneously, but after realizing the error, they should still have access to more quota quickly as the leak starts to drain the bucket.
    - The token bucket allows for sudden increase in traffic to some extent, while the leaky bucket is mainly used to ensure the smooth outflow rate.
* Cons:
    - When compared with token bucket, packet will be discarded instead of token.
    - The leaky bucket is normally implemented using a background process that simulates a leak. It looks for any active buckets that need to be drained, and drains each one in turn. The naive leaky bucket’s greatest weakness is its “drip” process. If it goes offline or gets to a capacity limit where it can’t drip all the buckets that need to be dripped, then new incoming requests might be limited incorrectly. There are a number of strategies to help avoid this danger, but if we could build an algorithm without a drip, it would be fundamentally more stable.

### Fixed window 
In a fixed window algorithm, a window size of n seconds (typically using human-friendly values, such as 60 or 3600 seconds) is used to track the rate. Each incoming request increments the counter for the window. If the counter exceeds a threshold, the request is discarded. The windows are typically defined by the floor of the current timestamp, so 12:00:03 with a 60 second window length, would be in the 12:00:00 window.

* Steps
    1. A window of size N is used to track the requests.
    2. There is one bucket for each of the unit time window.
    3. Each bucket maintains the count of number of requests in that particular window; If the counter exceeds a threshold, the request is discarded.

For example, a rate limiter for a service that allows only 10 requests per an hour will have the data model like below. Here, the buckets are the windows of one hour, with values storing the counts of the requests seen in that hour.

{
 "1AM-2AM": 7,
 "2AM-3AM": 8
}

With the current model in the above example, if a new request is seen at 2:45AM, we get the count from the current bucket(2AM-3AM) which is 8 and verify that if processing one more request exceeds the permissible limit of 10, if that is the case, an exception is raised; if not(which is the case here), count of the bucket is incremented by unit(to 9) and the request is allowed to be processed. Only the counts of the current window are stored and older windows are deleted when a new window is created(i.e in the above case, if the hour changes, older bucket is deleted)

Space Complexity: O(1) — Storing the current window count
Time Complexity: O(1) — Get and simple atomic increment operation

* Pros
    - It ensures recent requests get processed without being starved by old requests.
    - Can use inbuilt concurrency with redis like technologies
* Cons
    - Stamping elephant problem: A single burst of traffic that occurs near the boundary of a window can result in twice the rate of requests being processed, because it will allow requests for both the current and next windows within a short time.
    - If many consumers wait for a reset window, for example at the top of the hour, then they may stampede your API at the same time.
    - This in incorrect. Explanation: In the above case, if all the 7 requests in the 1AM-2AM bucket occurs from 1:30AM-2AM, and all the 8 requests from 2AM-3AM bucket occur from 2AM-2:30AM, then effectively we have 15(7 + 8) requests in the time range of 1:30AM-2:30AM, which is violating the condition of 10req/hr


### Sliding window logs
Ref: https://medium.com/@saisandeepmopuri/system-design-rate-limiter-and-data-modelling-9304b0d18250

Sliding Log rate limiting involves tracking a time stamped log for each consumer’s request. These logs are usually stored in a hash set or table that is sorted by time. Logs with timestamps beyond a threshold are discarded. When a new request comes in, we calculate the sum of logs to determine the request rate. If the request would exceed the threshold rate, then it is held.
* Steps
    1. For every user, a queue of timestamps representing the times at which all the historical calls have occurred within the timespan of recent most window is maintained.
    2. When ever a new request occurs, a check is made for any timestamps older than the window time and these are deleted as they are no longer relevant(Instead of doing this at every request, this step can also be done periodically after every ‘n’ mins or when a certain length of the queue is reached)
    3. The new timestamp is appended to the user’s queue
    4. If the number of elements in the queue is not more than that of the allowed count, the request is let through, else an exception is raised.

Space Complexity: O(Max requests seen in a window time) — Stores all the timestamps of the requests in a time window
Time Complexity: O(Max requests seen in a window time)— Deleting a subset of timestamps

* Pros
    - It does not suffer from the boundary conditions of fixed windows. The rate limit will be enforced precisely. - Since the sliding log is tracked for each consumer, you don’t have the stampede effect that challenges fixed windows
* Cons
	- High memory footprint. All the request timestamps needs to be maintained for a window time, thus requires lots of memory to handle multiple users or large window times
	- High time complexity for removing the older timestamps

#### Data Modelling
**In memory Design (Single machine with multiple threads)**

```
#  sliding-window-logs-in-memory.py
import time
import threading
from collections import deque

class RequestTimestamps(object):

	# lock is for concurrency in a multi threaded system
	# 100 req/min translates to requests = 100 and windowTimeInSec = 60
	def __init__(self, requests, windowTimeInSec):
		self.timestamps = deque()
		self.lock = threading.Lock()
		self.requests = requests
		self.windowTimeInSec = windowTimeInSec

	# eviction of timestamps older than the window time
	def evictOlderTimestamps(self, currentTimestamp):
		while len(self.timestamps) != 0 and (currentTimestamp - 
				self.timestamps[0] > self.windowTimeInSec):
			self.timestamps.popleft()

class SlidingWindowLogsRateLimiter(object):
	
	def __init__(self):
		self.lock = threading.Lock()
		self.ratelimiterMap = {}

	# Default of 100 req/minute
	# Add a new user with a request rate
	def addUser(self, userId, requests=100, windowTimeInSec=60):
		# hold lock to add in the user-metadata map
		with self.lock:
			if userId in self.ratelimiterMap:
				raise Exception("User already present")
			self.ratelimiterMap[userId] = RequestTimestamps(requests, windowTimeInSec)

	# Remove a user from the ratelimiter
	def removeUser(self, userId):
		with self.lock:
			if userId in self.ratelimiterMap:
				del self.ratelimiterMap[userId]

	# gives current time epoch in seconds
	@classmethod
	def getCurrentTimestampInSec(cls):
		return int(round(time.time()))

	# Checks if the service call should be allowed or not
	def shouldAllowServiceCall(self, userId):
		with self.lock:
			if userId not in self.ratelimiterMap:
				raise Exception("User is not present. Please white \
					list and register the user for service")
		userTimestamps = self.ratelimiterMap[userId]
		with userTimestamps.lock:
			currentTimestamp = self.getCurrentTimestampInSec()
			# remove all the existing older timestamps
			userTimestamps.evictOlderTimestamps(currentTimestamp)
			userTimestamps.timestamps.append(currentTimestamp)
			if len(userTimestamps.timestamps) > userTimestamps.requests:
				return False
			return True
```

The following solutions use redis based pipelines which provide ways to use optimistic locking in redis. One can go through redis transactions, to get a gist of concurrency in redis. The below data modelling is optional.

#### Sliding Window Logs — Redis Design

```
# sliding-window-logs-redis.py
import time
import redis

# redis connection
def get_connection(host="127.0.0.1", port="6379", db=0):
	connection = redis.StrictRedis(host=host, port=port, db=db)
	return connection

class SlidingWindowLogRatelimiter(object):

	"""
	representation of data stored in redis
	metadata
	--------
	"userid_metadata": {
            "requests": 2,
	    "window_time": 30
	}
	
	timestamps
	----------
	"userid_timestamps": sorted_set([
	    "ts1": "ts1",
	    "ts2": "ts2"
	])
	"""
	REQUESTS = "requests"
	WINDOW_TIME = "window_time"
	METADATA_SUFFIX = "_metadata"
	TIMESTAMPS = "_timestamps"
	INF = 9999999999

	def __init__(self):
		self.con = get_connection()

	# timestamp in seconds
	@classmethod
	def getCurrentTimestampInSec(cls):
		return int(round(time.time()))

	# Adds a new user's rate of requests to be allowed
	def addUser(self, userId, requests=100, windowTimeInSec=60):
		self.con.hmset(userId + self.METADATA_SUFFIX, {
			self.REQUESTS: requests,
			self.WINDOW_TIME: windowTimeInSec
		})

	# get the user metadata storing the number of requests per window time
	def getRateForUser(self, userId):
		val = self.con.hgetall(userId + self.METADATA_SUFFIX)
		if val is None:
			raise Exception("Un-registered user: " + userId)
		return int(val[self.REQUESTS]), int(val[self.WINDOW_TIME])

	# Removes a user's metadata and timestamps
	def removeUser(self, userId):
		self.con.delete(userId + self.METADATA_SUFFIX, userId + self.TIMESTAMPS)

	# Atomically add an element to the timestamps and return the total number of requests
	# in the current window time. 
	def addTimeStampAtomicallyAndReturnSize(self, userId, timestamp):
		# Transaction holds an optimistic lock over the redis entries userId + self.METADATA_SUFFIX 
		# and userId + self.TIMESTAMPS. The changes in _addNewTimestampAndReturnTotalCount
		# are committed only if none of these entries get changed through out
		_, size = self.con.transaction(
			lambda pipe: self._addNewTimestampAndReturnTotalCount(userId, timestamp, pipe),
			userId + self.METADATA_SUFFIX, userId + self.TIMESTAMPS
		)
		return size
	
	def _addNewTimestampAndReturnTotalCount(self, userId, timestamp, redisPipeline):
		# A two element array with first one representing success of adding an element into
		# sorted set and other as the count of the sorted set is returned by this method
		redisPipeline.multi()
		redisPipeline.zadd(userId + self.TIMESTAMPS, timestamp, timestamp)
		redisPipeline.zcount(userId + self.TIMESTAMPS, 0, self.INF)

	# decide to allow a service call or not
	# we use sorted sets datastructure in redis for storing our timestamps. 
	# For more info, visit https://redis.io/topics/data-types
	def shouldAllowServiceCall(self, userId):
		maxRequests, unitTime = self.getRateForUser(userId)
		currentTimestamp = self.getCurrentTimestampInSec()
		# evict older entries 
		oldestPossibleEntry = currentTimestamp - unitTime
		# removes all the keys from start to oldest bucket
		self.con.zremrangebyscore(userId + self.TIMESTAMPS, 0, oldestPossibleEntry)
		currentRequestCount = self.addTimeStampAtomicallyAndReturnSize(
			userId, currentTimestamp
		)
		print currentRequestCount, maxRequests
		if currentRequestCount > maxRequests:
			return False
		return True
```

### Sliding window counters
* This is a hybrid of Fixed Window Counters and Sliding Window logs
* The entire window time is broken down into smaller buckets. The size of each bucket depends on how much elasticity is allowed for the rate limiter
* Each bucket stores the request count corresponding to the bucket range.

For example, in order to build a rate limiter of 100 req/hr, say a bucket size of 20 mins is chosen, then there are 3 buckets in the unit time

For a window time of 2AM to 3AM, the buckets are

{
 "2AM-2:20AM": 10,
 "2:20AM-2:40AM": 20,
 "2:40AM-3:00AM": 30
}

If a request is received at 2:50AM, we find out the total requests in last 3 buckets including the current and add them, in this case they sum upto 60 (<100), so a new request is added to the bucket of 2:40AM–3:00AM giving...

{
 "2AM-2:20AM": 10,
 "2:20AM-2:40AM": 20,
 "2:40AM-3:00AM": 31
}

**Note:** This is not a completely correct, for example: At 2:50, a time interval from 1:50 to 2:50 should be considered, but in the above example the first 10 mins isn’t considered and it may happen that in this missed 10 mins, there might’ve been a traffic spike and the request count might be 100 and hence the request is to be rejected. But by tuning the bucket size, we can reach a fair approximation of the ideal rate limiter.

Space Complexity: O(number of buckets)
Time Complexity: O(1) — Fetch the recent bucket, increment and check against the total sum of buckets(can be stored in a totalCount variable).

* Pros
	- No large memory footprint as only the counts are stored

* Cons
	- Works only for not-so-strict look back window times, especially for smaller unit times

#### Data Modelling
**In memory Design (Single machine with multiple threads)**

```
# sliding-window-counters-in-memory.py
import time
import threading

class RequestCounters(object):
	# Every window time is broken down to 60 parts
	# 100 req/min translates to requests = 100 and windowTimeInSec = 60
	def __init__(self, requests, windowTimeInSec, bucketSize=10):
		self.counts = {}
		self.totalCounts = 0
		self.requests = requests
		self.windowTimeInSec = windowTimeInSec
		self.bucketSize = bucketSize
		self.lock = threading.Lock()

	# Gets the bucket for the timestamp
	def getBucket(self, timestamp):
		factor = self.windowTimeInSec / self.bucketSize
		return (timestamp // factor) * factor

	# Gets the bucket list corresponding to the current time window
	def _getOldestvalidBucket(self, currentTimestamp):
		return self.getBucket(currentTimestamp - self.windowTimeInSec)

	# Remove all the older buckets that are not relevant anymore
	def evictOlderBuckets(self, currentTimestamp):
		oldestValidBucket = self._getOldestvalidBucket(currentTimestamp)
		bucketsToBeDeleted = filter(
			lambda bucket: bucket < oldestValidBucket, self.counts.keys())
		for bucket in bucketsToBeDeleted:
			bucketCount = self.counts[bucket]
			self.totalCounts -= bucketCount
			del self.counts[bucket]

class SlidingWindowCounterRateLimiter(object):
	def __init__(self):
		self.lock = threading.Lock()
		self.ratelimiterMap = {}

	# Default of 100 req/minute
	# Add a new user with a request rate
	# If a request from un-registered user comes, we throw an Exception
	def addUser(self, userId, requests=100, windowTimeInSec=60):
		with self.lock:
			if userId in self.ratelimiterMap:
				raise Exception("User already present")
			self.ratelimiterMap[userId] = RequestCounters(requests, windowTimeInSec)

	def removeUser(self, userId):
		with self.lock:
			if userId in self.ratelimiterMap:
				del self.ratelimiterMap[userId]

	@classmethod
	def getCurrentTimestampInSec(cls):
		return int(round(time.time()))

	def shouldAllowServiceCall(self, userId):
		with self.lock:
			if userId not in self.ratelimiterMap:
				raise Exception("User is not present")
		userTimestamps = self.ratelimiterMap[userId]
		with userTimestamps.lock:
			currentTimestamp = self.getCurrentTimestampInSec()
			# remove all the existing older timestamps
			userTimestamps.evictOlderBuckets(currentTimestamp)
			currentBucket = userTimestamps.getBucket(currentTimestamp)
			userTimestamps.counts[currentBucket] = userTimestamps.counts.
				get(currentBucket, 0) + 1
			userTimestamps.totalCounts += 1
			if userTimestamps.totalCounts > userTimestamps.requests:
				return False
			return True
```
**Sliding Window Counters — Redis Design**

```
# sliding-window-counters-redis.py 
import time
import redis

# redis connection
def get_connection(host="127.0.0.1", port="6379", db=0):
	connection = redis.StrictRedis(host=host, port=port, db=db)
	return connection

class SlidingWindowCounterRateLimiter(object):

	"""
	representation of data stored in redis
	metadata
	--------
	"userid_metadata": {
            "requests": 2,
	    "window_time": 30
	}
	
	counts
	-------
	"userid_counts": {
	    "bucket1": 2,
	    "bucket2": 3
	}
	"""
	REQUESTS = "requests" # key in the metadata representing the max number of requests
	WINDOW_TIME = "window_time" # key in the metadata representing the window time
	METADATA_SUFFIX = "_metadata" # metadata suffix
	COUNTS = "_counts" # count buckets suffix
	
	def __init__(self, bucketSize=10):
		# bucket size can be coarser than 10 sec based on the window size.
		self.bucketSize = bucketSize # in seconds
		self.con = get_connection()

	# current timestamp in seconds.
	@classmethod
	def getCurrentTimestampInSec(cls):
		return int(round(time.time()))	

	def getBucket(self, timestamp, windowTimeInSec):
		factor = windowTimeInSec / self.bucketSize
		return (timestamp // factor) * factor

	# Adds a new user's rate of requests to be allowed.
	# using redis hashes to store the user metadata.
	def addUser(self, userId, requests=100, windowTimeInSec=60):
		# TODO: Make sure that the given windowTimeInSec is a multiple of bucketSize
		self.con.hmset(userId + self.METADATA_SUFFIX, {
			self.REQUESTS: requests,
			self.WINDOW_TIME: windowTimeInSec
		})

	# Get the user metadata storing the number of requests per window time.
	def getRateForUser(self, userId):
		val = self.con.hgetall(userId + self.METADATA_SUFFIX)
		if val is None:
			raise Exception("Un-registered user: " + userId)
		return int(val[self.REQUESTS]), int(val[self.WINDOW_TIME])
	
	# Removes a user's metadata and timestamps.
	def removeUser(self, userId):
		self.con.delete(userId + self.METADATA_SUFFIX, userId + self.COUNTS)

	# Atomically increments hash key val by unit and returns. Uses optimistic locking
	# over userId + self.COUNTS redis key.
	def _incrementAHashKeyValByUnitAmotically(self, userId, bucket, redisPipeline):
		# A two element array with first one representing success of updating the
		# bucket value and other giving a list of all the values(counts) of the buckets.
		count = redisPipeline.hmget(userId + self.COUNTS, bucket)[0]
		if count is None:
			count = 0
		currentBucketCount = int(count)
		redisPipeline.multi()
		redisPipeline.hmset(userId + self.COUNTS, {bucket: currentBucketCount + 1})
		redisPipeline.hvals(userId + self.COUNTS)	

	# Deciding if the rate has been crossed.
	# we're using redis hashes to store the counts.
	def shouldAllowServiceCall(self, userId):
		allowedRequests, windowTime = self.getRateForUser(userId)
		# evict older entries
		allBuckets = map(int, self.con.hkeys(userId + self.COUNTS))
		currentTimestamp = self.getCurrentTimestampInSec()
		oldestPossibleEntry = currentTimestamp - windowTime
		bucketsToBeDeleted = filter(
			lambda bucket: bucket < oldestPossibleEntry, allBuckets)
		if len(bucketsToBeDeleted) != 0:
			self.con.hdel(userId + self.COUNTS, *bucketsToBeDeleted)
		currentBucket = self.getBucket(currentTimestamp, windowTime)
		# transaction holds an optimistic lock over the redis entries
		# userId + self.METADATA_SUFFIX, userId + self.COUNTS.
		# The changes in _incrementAHashKeyValByUnitAmotically are committed only 
		# if none of these entries get changed.
		_, requests = self.con.transaction(
		    lambda pipe: self.
			_incrementAHashKeyValByUnitAmotically(userId, currentBucket, pipe),
    		    userId + self.COUNTS, userId + self.METADATA_SUFFIX
		)
		if sum(map(int, requests)) > allowedRequests:
			return False
		return True
```
References: https://blog.figma.com/an-alternative-approach-to-rate-limiting-f8a06cf7c94c

## Rate Limiting in Distributed Systems
### Synchronization Policies

If you want to enforce a global rate limit when you are using a cluster of multiple nodes, you must set up a policy to enhttps://konghq.com/blog/using-instaclustr-and-cassandra-with-kong/force it. If each node were to track its own rate limit, then a consumer could exceed a global rate limit when requests are sent to different nodes. In fact, the greater the number of nodes, the more likely the user will be able to exceed the global limit.

The simplest way to enforce the limit is to set up sticky sessions in your load balancer so that each consumer gets sent to exactly one node. The disadvantages include a lack of fault tolerance and scaling problems when nodes get overloaded.

A better solution that allows more flexible load-balancing rules is to use a centralized data store such as Redis or Cassandra. This will store the counts for each window and consumer. The two main problems with this approach are increased latency making requests to the data store, and race conditions, which we will discuss next.

![API Rate Limiter](06-rate-limit-kong.png)


### Race Conditions

One of the largest problems with a centralized data store is the potential for race conditions in high concurrency request patterns. This happens when you use a naïve “get-then-set” approach, wherein you retrieve the current rate limit counter, increment it, and then push it back to the datastore. The problem with this model is that in the time it takes to perform a full cycle of read-increment-store, additional requests can come through, each attempting to store the increment counter with an invalid (lower) counter value. This allows a consumer sending a very high rate of requests to bypass rate limiting controls.

One way to avoid this problem is to put a “lock” around the key in question, preventing any other processes from accessing or writing to the counter. This would quickly become a major performance bottleneck, and does not scale well, particularly when using remote servers like Redis as the backing datastore.

A better approach is to use a “set-then-get” mindset, relying on atomic operators that implement locks in a very performant fashion, allowing you to quickly increment and check counter values without letting the atomic operations get in the way.

![API Rate Limiter](06-2-rate-limit-kong.png)

### Optimizing for Performance

The other disadvantage of using a centralized data store is increased latency when checking on the rate limit counters. Unfortunately, even checking a fast data store like Redis would result in milliseconds of additional latency for every request.

In order to make these rate limit determinations with minimal latency, it’s necessary to make checks locally in memory. This can be done by relaxing the rate check conditions and using an eventually consistent model. For example, each node can create a data sync cycle that will synchronize with the centralized data store. Each node periodically pushes a counter increment for each consumer and window it saw to the datastore, which will atomically update the values. The node can then retrieve the updated values to update it’s in-memory version. This cycle of converge → diverge → reconverge among nodes in the cluster is eventually consistent.

![API Rate Limiter](07-rate-limit-kong.png)

The periodic rate at which nodes converge should be configurable. Shorter sync intervals will result in less divergence of data points when traffic is spread across multiple nodes in the cluster (e.g., when sitting behind a round robin balancer), whereas longer sync intervals put less read/write pressure on the datastore, and less overhead on each node to fetch new synced values.


## Single machine rate limit

### Guava rate limiter 
* Implemented on top of token bucket. It has two implementations:
* SmoothBursty / SmoothWarmup (The RateLimiterSmoothWarmingUp method has a warm-up period after teh startup. It gradually increases the distribution rate to the configured value. This feature is suitable for scenarios where the system needs some time to warm up after startup.)

#### Implementation  
##### Producer consumer pattern
* Def: Use a producer thread to add token, the thread who uses rate limiter act as consumer.
* Cons:
  - High cost for maintaining so many threads: Suppose use server cron timer as producer to add token. Suppose the goal is to rate limit on user visiting frequency andd there are 6 million users, then 6 million cron functionality needs to be created.
  - Rate limiting are usually used under high server loads. During such peak traffic time the server timer might not be that accurate and reliable.

#### Record the next time a token is available
* Each time a token is expected, first take from the storedPermits; If not enough, then compare against nextFreeTicketMicros (update simultaneously using resync function) to see whether freshly generated tokens could satisfy the requirement. If not, sleep until nextFreeTicketMicros to acquire the next available fresh token.

```
// The number of currently stored tokens
double storedPermits;
// The maximum number of stored tokens
double maxPermits;
// The interval to add tokens
double stableIntervalMicros;
/**
 * The time for the next thread to call the acquire() method
 * RateLimiter allows preconsumption. After a thread preconsumes any tokens,
 the next thread needs to wait until nextFreeTicketMicros to acquire tokens.
 */
private long nextFreeTicketMicros = 0L;
```

```
/**
 * Updates {@code storedPermits} and {@code nextFreeTicketMicros} based on the current time.
 */
void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
      storedPermits = min(maxPermits, storedPermits + newPermits);
      nextFreeTicketMicros = nowMicros;
    }
}
```
##### Warm up feature
* Motivation: How to gracefully deal past underutilization
  - Past underutilization could mean that excess resources are available. Then, the RateLimiter should speed up for a while, to take advantage of these resources. This is important when the rate is applied to networking (limiting bandwidth), where past underutilization typically translates to "almost empty buffers", which can be filled immediately.
  - Past underutilization could mean that "the server responsible for handling the request has become less ready for future requests", i.e. its caches become stale, and requests become more likely to trigger expensive operations (a more extreme case of this example is when a server has just booted, and it is mostly busy with getting itself up to speed).

* Implementation
   - When the RateLimiter is not used, this goes right (up to maxPermits)
   - When the RateLimiter is used, this goes left (down to zero), since if we have storedPermits, we serve from those first
   - When _unused_, we go right at a constant rate! The rate at which we move to the right is chosen as maxPermits / warmupPeriod. This ensures that the time it takes to go from 0 to maxPermits is equal to warmupPeriod.
   - When _used_, the time it takes, as explained in the introductory class note, is equal to the integral of our function, between X permits and X-K permits, assuming we want to spend K saved permits.

```
             ^ throttling
             |
       cold  +                  /
    interval |                 /.
             |                / .
             |               /  .   ← "warmup period" is the area of the trapezoid between
             |              /   .     thresholdPermits and maxPermits
             |             /    .
             |            /     .
             |           /      .
      stable +----------/  WARM .
    interval |          .   UP  .
             |          . PERIOD.
             |          .       .
           0 +----------+-------+--------------→ storedPermits
             0 thresholdPermits maxPermits
```

* References
    1. https://segmentfault.com/a/1190000012875897?spm=a2c65.11461447.0.0.74817a50Dt3FUO
    2. https://www.alibabacloud.com/blog/detailed-explanation-of-guava-ratelimiters-throttling-mechanism_594820

### Ratelimiter within Resiliency4J
* https://dzone.com/articles/rate-limiter-internals-in-resilience4j
* https://blog.csdn.net/mickjoust/article/details/102411585

## Distributed rate limit
### Sticky sessions
- The simplest way to enforce the limit is to set up sticky sessions in your load balancer so that each consumer gets sent to exactly one node. The disadvantages include a lack of fault tolerance and scaling problems when nodes get overloaded.

### Nginx based rate limiting

### Redis based rate limiter
* Use a centralized data store such as Redis to store the counts for each window and consumer.

#### Implementation
##### Sliding log implementation using ZSet
* See [Dojo engineering blog for details](https://engineering.classdojo.com/blog/2015/02/06/rolling-rate-limiter/)
    1. Each identifier/user corresponds to a sorted set data structure. The keys and values are both equal to the (microsecond) times at which actions were attempted, allowing easy manipulation of this list.
    2. When a new action comes in for a user, all elements in the set that occurred earlier than (current time - interval) are dropped from the set.
    3. If the number of elements in the set is still greater than the maximum, the current action is blocked.
    4. If a minimum difference has been set and the most recent previous element is too close to the current time, the current action is blocked.
    5. The current action is then added to the set.
    6. Note: if an action is blocked, it is still added to the set. This means that if a user is continually attempting actions more quickly than the allowed rate, all of their actions will be blocked until they pause or slow their requests.
    7. If the limiter uses a redis instance, the keys are prefixed with namespace, allowing a single redis instance to support separate rate limiters.
    8. All redis operations for a single rate-limit check/update are performed as an atomic transaction, allowing rate limiters running on separate processes or machines to share state safely.

##### Sliding window implementation
* https://blog.callr.tech/rate-limiting-for-distributed-systems-with-redis-and-lua/
* https://github.com/wangzheng0822/ratelimiter4j

##### Token bucket implementation
* https://github.com/vladimir-bukhtoyarov/bucket4j

#### Challenges
##### How to handle race conditions
1. One way to avoid this problem is to put a “lock” around the key in question, preventing any other processes from accessing or writing to the counter. This would quickly become a major performance bottleneck, and does not scale well, particularly when using remote servers like Redis as the backing datastore.
2. A better approach is to use a “set-then-get” mindset, relying on Redis' atomic operators that implement locks in a very performant fashion, allowing you to quickly increment and check counter values without letting the atomic operations get in the way.
3. Use Lua scripts for atomic and better performance.

##### How to handle the additional latency introduce by performance
1. In order to make these rate limit determinations with minimal latency, it’s necessary to make checks locally in memory. This can be done by relaxing the rate check conditions and using an eventually consistent model. For example, each node can create a data sync cycle that will synchronize with the centralized data store.
2. Each node periodically pushes a counter increment for each consumer and window it saw to the datastore, which will atomically update the values. The node can then retrieve the updated values to update it’s in-memory version. This cycle of converge → diverge → reconverge among nodes in the cluster is eventually consistent.
  - https://konghq.com/blog/how-to-design-a-scalable-rate-limiting-algorithm/
  -

##### How to avoid multiple round trips for different buckets:
* Use Redis Pipeline to combine the INCRE and EXPIRE commands
* If using N multiple bucket sizes, still need N round trips to Redis.
  - TODO: Could we also combine different bucket size together? How will the result for multiple results being passed back from Redis pipeline
* [Redis rate limiter implementation in python](https://www.binpress.com/rate-limiting-with-redis-1/)

##### Performance bottleneck and single point failure due to Redis
* Solution: ??

##### Static rate limit threshold
* Concurrency rate limit
  - Netflix Concurrency Limits: https://github.com/Netflix/concurrency-limits
  - Resiliency 4j said no for cache-based distributed rate limit: https://github.com/resilience4j/resilience4j/issues/350
  - Resiliency 4j adaptive capacity management: https://github.com/resilience4j/resilience4j/issues/201

### Ratelimiter within CloudBouncer
* Use gossip protocol to sync redis counters
  - https://yahooeng.tumblr.com/post/111288877956/cloud-bouncer-distributed-rate-limiting-at-yahoo

### Redis cell rate limiter
* An advanced version of GRCA algorithm
* References
    - You could find the intuition on https://jameslao.com/post/gcra-rate-limiting/
    - It is implemented in Rust because it offers more memory security. https://redislabs.com/blog/redis-cell-rate-limiting-redis-module/
                                                                                                                                                                                                               246,1         Bot


# Scaling your API with rate limiter

Rate limiting is a common technique used to improve the security and durability of a web application.

For example, a simple script can make thousands of web requests per second. Whether malicious, apathetic, or just a bug, your application and infrastructure may not be able to cope with the load. For more details, see Denial-of-service attack. Most cases can be mitigated by limiting the rate of requests from a single IP address.

Most brute-force attacks are similarly mitigated by a rate limit.


Availability and reliability are paramount for all web applications and APIs. If you’re providing an API, chances are you’ve already experienced sudden increases in traffic that affect the quality of your service, potentially even leading to a service outage for all your users.

The first few times this happens, it’s reasonable to just add more capacity to your infrastructure to accommodate user growth. However, when you’re running a production API, not only do you have to make it robust with techniques like idempotency, you also need to build for scale and ensure that one bad actor can’t accidentally or deliberately affect its availability.

Rate limiting can help make your API more reliable in the following scenarios:

* One of your users is responsible for a spike in traffic, and you need to stay up for everyone else.
* One of your users has a misbehaving script which is accidentally sending you a lot of requests. Or, even worse, one of your users is intentionally trying to overwhelm your servers.
* A user is sending you a lot of lower-priority requests, and you want to make sure that it doesn’t affect your high-priority traffic. For example, users sending a high volume of requests for analytics data could affect critical transactions for other users.
* Something in your system has gone wrong internally, and as a result you can’t serve all of your regular traffic and need to drop low-priority requests.

At Stripe, we’ve found that carefully implementing a few rate limiting strategies helps keep the API available for everyone. In this post, we’ll explain in detail which rate limiting strategies we find the most useful, how we prioritize some API requests over others, and how we started using rate limiters safely without affecting our existing users’ workflows.

## Rate limiters and load shedders

A rate limiter is used to control the rate of traffic sent or received on the network. **When should you use a rate limiter?** If your users can afford to change the pace at which they hit your API endpoints without affecting the outcome of their requests, then a rate limiter is appropriate. If spacing out their requests is not an option (typically for real-time events), then you’ll need another strategy outside the scope of this post (most of the time you just need more infrastructure capacity).

Our users can make a lot of requests: for example, batch processing payments causes sustained traffic on our API. We find that clients can always (barring some extremely rare cases) spread out their requests a bit more and not be affected by our rate limits.

Rate limiters are amazing for day-to-day operations, but during incidents (for example, if a service is operating more slowly than usual), we sometimes need to drop low-priority requests to make sure that more critical requests get through. This is called load shedding. It happens infrequently, but it is an important part of keeping Stripe available.

A load shedder makes its decisions based on the whole state of the system rather than the user who is making the request. Load shedders help you deal with emergencies, since they keep the core part of your business working while the rest is on fire.

Using different kinds of rate limiters in concert

Once you know rate limiters can improve the reliability of your API, you should decide which types are the most relevant.

At Stripe, we operate 4 different types of limiters in production. The first one, the Request Rate Limiter, is by far the most important one. We recommend you start here if you want to improve the robustness of your API.

## Request rate limiter

This rate limiter restricts each user to N requests per second. Request rate limiters are the first tool most APIs can use to effectively manage a high volume of traffic.

Our rate limits for requests is constantly triggered. It has rejected millions of requests this month alone, especially for test mode requests where a user inadvertently runs a script that’s gotten out of hand.

Our API provides the same rate limiting behavior in both test and live modes. This makes for a good developer experience: scripts won't encounter side effects due to a particular rate limit when moving from development to production.

After analyzing our traffic patterns, we added the ability to briefly burst above the cap for sudden spikes in usage during real-time events (e.g. a flash sale.)


Request rate limiters restrict users to a maximum number of requests per second.

Concurrent requests limiter

Instead of “You can use our API 1000 times a second”, this rate limiter says “You can only have 20 API requests in progress at the same time”. Some endpoints are much more resource-intensive than others, and users often get frustrated waiting for the endpoint to return and then retry. These retries add more demand to the already overloaded resource, slowing things down even more. The concurrent rate limiter helps address this nicely.

Our concurrent request limiter is triggered much less often (12,000 requests this month), and helps us keep control of our CPU-intensive API endpoints. Before we started using a concurrent requests limiter, we regularly dealt with resource contention on our most expensive endpoints caused by users making too many requests at one time. The concurrent request limiter totally solved this.

It is completely reasonable to tune this limiter up so it rejects more often than the Request Rate Limiter. It asks your users to use a different programming model of “Fork off X jobs and have them process the queue” compared to “Hammer the API and back off when I get a HTTP 429”. Some APIs fit better into one of those two patterns so feel free to use which one is most suitable for the users of your API.


Concurrent request limiters manage resource contention for CPU-intensive API endpoints.

Fleet usage load shedder

Using this type of load shedder ensures that a certain percentage of your fleet will always be available for your most important API requests.

We divide up our traffic into two types: critical API methods (e.g. creating charges) and non-critical methods (e.g. listing charges.) We have a Redis cluster that counts how many requests we currently have of each type.

We always reserve a fraction of our infrastructure for critical requests. If our reservation number is 20%, then any non-critical request over their 80% allocation would be rejected with status code 503.

We triggered this load shedder for a very small fraction of requests this month. By itself, this isn’t a big deal—we definitely had the ability to handle those extra requests. But we’ve had other months where this has prevented outages.


Fleet usage load shedders reserves fleet resources for critical requests.

Worker utilization load shedder

Most API services use a set of workers to independently respond to incoming requests in a parallel fashion. This load shedder is the final line of defense. If your workers start getting backed up with requests, then this will shed lower-priority traffic.

This one gets triggered very rarely, only during major incidents.

We divide our traffic into 4 categories:

Critical methods
POSTs
GETs
Test mode traffic
We track the number of workers with available capacity at all times. If a box is too busy to handle its request volume, it will slowly start shedding less-critical requests, starting with test mode traffic. If shedding test mode traffic gets it back into a good state, great! We can start to slowly bring traffic back. Otherwise, it’ll escalate and start shedding even more traffic.

It’s very important that shedding and bringing load happen slowly, or you can end up flapping (“I got rid of testmode traffic! Everything is fine! I brought it back! Everything is awful!”). We used a lot of trial and error to tune the rate at which we shed traffic, and settled on a rate where we shed a substantial amount of traffic within a few minutes.

Only 100 requests were rejected this month from this rate limiter, but in the past it’s done a lot to help us recover more quickly when we have had load problems. This load shedder limits the impact of incidents that are already happening and provides damage control, while the first three are more preventative.


Worker utilization load shedders reserve workers for critical requests.

Building rate limiters in practice

Now that we’ve outlined the four basic kinds of rate limiters we use and what they’re for, let’s talk about their implementation. What rate limiting algorithms are there? How do you actually implement them in practice?

We use the token bucket algorithm to do rate limiting. This algorithm has a centralized bucket host where you take tokens on each request, and slowly drip more tokens into the bucket. If the bucket is empty, reject the request. In our case, every Stripe user has a bucket, and every time they make a request we remove a token from that bucket.

We implement our rate limiters using Redis. You can either operate the Redis instance yourself, or, if you use Amazon Web Services, you can use a managed service like ElastiCache.

Here are important things to consider when implementing rate limiters:

Hook the rate limiters into your middleware stack safely. Make sure that if there were bugs in the rate limiting code (or if Redis were to go down), requests wouldn’t be affected. This means catching exceptions at all levels so that any coding or operational errors would fail open and the API would still stay functional.
Show clear exceptions to your users. Figure out what kinds of exceptions to show your users. In practice, you should decide if you want HTTP 429 (Too Many Requests) or HTTP 503 (Service Unavailable) and what is the most accurate depending on the situation. The message you return should also be actionable.
Build in safeguards so that you can turn off the limiters. Make sure you have kill switches to disable the rate limiters should they kick in erroneously. Having feature flags in place can really help should you need a human escape valve. Set up alerts and metrics to understand how often they are triggering.
Dark launch each rate limiter to watch the traffic they would block. Evaluate if it is the correct decision to block that traffic and tune accordingly. You want to find the right thresholds that would keep your API up without affecting any of your users’ existing request patterns. This might involve working with some of them to change their code so that the new rate limit would work for them.
Conclusion

Rate limiting is one of the most powerful ways to prepare your API for scale. The different rate limiting strategies described in this post are not all necessary on day one, you can gradually introduce them once you realize the need for rate limiting.

Our recommendation is to follow the following steps to introduce rate limiting to your infrastructure:

Start by building a Request Rate Limiter. It is the most important one to prevent abuse, and it’s by far the one that we use the most frequently.
Introduce the next three types of rate limiters over time to prevent different classes of problems. They can be built slowly as you scale.
Follow good launch practices as you're adding new rate limiters to your infrastructure. Handle any errors safely, put them behind feature flags to turn them off easily at any time, and rely on very good observability and metrics to see how often they’re triggering.
