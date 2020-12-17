# CricInfo / CricBuzz System design

## Summary

*    The scores are stored in the memory storage called databases. Each and every entry will be unique. Each database will have further partitions called tables which each can contain like bowling details, batting details, strike rates, run rates etc.,
 
*   This data is extracted using back end (server side) programming language. Generally these languages are Python, PHP etc., For example Quora uses Python as its back end language.

*    Now the extracted data is displayed using a front end (client side) language on the user interface. Generally this is HTML along with CSS and Javascript.

*    Now if a new score is entered into database, you have to refresh the web page to get new score details. But this refreshing can be avoided using some http attributes. These attributes are used to refresh the web page automatically for every particular time interval like 1s, 5s etc.,
 
*   Now if you want to get some data from database without refreshing, we can use technologies like Ajax which uses Javascript along with XML or JSON which are used to represent data.


*TRAFFIC:* Before designing a system design for CRICINFO we need to know the traffic. you can assume or go to

https://www.similarweb.com/website/cricinfo.com/#overview or Alexa Website Traffic(https://www.alexa.com/siteinfo), Statistics and Analytics
so approximately 2.5M visits every day.


## How do scores are imported to Cricinfo’s system?

* Its a combination of automatic and manual.

* There is a scorer and a commentator, often in the same cabin, who input that data. Usually, the scorer inputs the runs scored off the ball, whether any extras were conceded or not, where the ball was played (on the ground) etc. The scorer also inputs the next batsman when he walks in, the mode of dismissal, the bowler who comes into bowl
    Or the scores are imported directly from Stadium scoreboard software by push or pull or socket.

## Product APIS

* Cricinfo/Cricbuzz websites not only consume data for their sites, but they also feed data to vendors/Third parties like apps like Games, other sports websites and TV channels etc.

### What are the information which they sell/share?
*    Athletes: Biographical, profile, and statistical data for sports’ biggest stars, covering all major sports
*    LIVE SCORES
*    Calendar
*    headlines
*    leaderboards
*    Teams
*    photos etc

As these APIs are paid the service provider should prioritize the traffic and also sometimes customizable and high availability.

## Scalable APIs :

There are so many vendors, everyone has different/custom feeds/API requirements but writing those many APIs is the case. So we need to have generic API s

Cricinfo has two kinds of APIS

*    Core APIS
*    Product API

Most of the time a single product API is a combination of many core APIs, in that case, Binder will call the core APIs in parallel and gets the product API ready.

Binder is a component which handles calling core APIs (asynchronously) and merging and returning to product API.

### What if one of core API call fails while calling Asynchronously?

### Make the main call/Product API call also fail !!

### Why? To have predictable output. as we can’t send incomplete data to third-party as third-party consumers might have not handled the scenario of incomplete/partial data

## Content search engine:

Used to sort the different content like Editorial content, WordPress content, scores, highlights etc based on whats most relevant based on the past 24 hours. which returns the relevant content to show to users.


## Cricinfo uses 3 tiers caching!

1. Local cache to save hot/famous APIs and its local because to avoid network calls to distributed cache(EH cache)

2. If the data is not found in the Local cache then go to Distributed cache(Memcache)

3. Varnish between the binder and Core APIS to avoid stampede effect. eg: when about 30 to 40 calls for the same API are coming to varnish, it combines all of that API calls and get it once and cache it for few seconds also

This stampede can be avoided by Request collapse : serve response to all similar request in a few millisecond window so everyone gets the response and once this response is cached in Varnish.

## Hystrix:

Hystrix is a latency and fault tolerance library designed to isolate points of access to remote systems, services and 3rd party libraries, stop cascading failure and enable resilience in complex distributed systems where failure is inevitable.

It is used to do Circuit breaking for all of the Microservices(Core apis and product APIs) used.

## REAL-TIME DATA UPDATE USING REST APIS!!

As sports fans need data in real-time there should be a way to push data to clients

Cricinfo has capabilities to send 1 Million WebSocket Updates to Fans in less than 100MS.

How do they do it? they use Redis queues which are mapped to each match which is happening at day, any updates related to that match will be posted to that queue.

*    A scheduler will keep on querying the scoring and match statistics API
    (Poll couple of times(2–3 times) every second to check if there is change in the API data, if there is then send it to next stage)
*    Then the data will be sent to check the difference of the data sent previously VS current to reduce the payload size. only the new information will be sent to respective Queue
*    WebSocket connection from the client to WebSocket server FANOUT data which is Redis queue to clients.

For old games which already happened you never need to use Websockets as the data is readliy available and can be queried via APIS

## How to do API monitoring

OPENTSDB: Scalable Time Series Database Store and serve massive amounts of time series data without losing granularity.

    Runs on Hadoop and HBase
    Scales to millions of writes per second
    Add capacity by adding nodes
    Generate graphs from the GUI
    Pull from the HTTP API
    Supports Graphana for visualization of data

## Graphana:

Grafana allows you to query, visualize, alert on and understand your metrics no matter where they are stored. Create, explore, and share dashboards with your team and foster a data-driven culture.

## Databases:

MongoDB

Couch DB

Cassandra

Riak

Membase

Redis

Ref: https://medium.com/@narengowda/cricinfo-cricbuzz-system-design-c596e8506669

