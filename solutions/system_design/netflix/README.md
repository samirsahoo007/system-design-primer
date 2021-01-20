# Design YouTube/Netflix (a global video streaming service)

Netflix operates in two clouds: AWS and Open Connect.

Both clouds must work together seamlessly to deliver endless hours of customer-pleasing video.

Netflix has 3 main components which we are going to discuss today

* OC or netflix CDN
* backend
* client

Lets first talk about some high level working of Netflix and then jump in to these 3 components.

The client is the user interface on any device used to browse and play Netflix videos. TV, XBOX, laptop or mobile phone etc

Anything that doesn't involve serving video is handled in AWS.

Everything that happens after you hit play is handled by Open Connect. Open Connect is Netflix’s custom global content delivery network (CDN).

**Open Connect** stores Netflix video in different locations throughout the world. When you press play the video streams from Open Connect, into your device, and is displayed by the client.

**CDN** — A content delivery network (CDN) is a system of distributed servers (network) that deliver pages and other Web content to a user, based on the geographic locations of the user, the origin of the webpage and the content delivery server.

## How Netflix onboard a movie/video:
Before this movie is made available to users, Netflix must convert the video into a format that works best for your device. This process is called transcoding or encoding.

**Transcoding** is the process that converts a video file from one format to another, to make videos viewable across different platforms and devices.

Whys do we need to do it? why can't we just play the source video?

The original movie/video comes in a high definition format that’s many terabytes in size. Also, Netflix supports 2200 different devices. Each device has a video format that looks best on that particular device. If you’re watching Netflix on an iPhone, you’ll see a video that gives you the best viewing experience on the iPhone.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/netflix_transcoding.png)

Netflix also creates files optimized for different network speeds. If you’re watching on a fast network, you’ll see the higher quality video than you would if you’re watching over a slow network. And also depends on your Netflix plan. that said Netflix does create approx 1,200 files for every movie !!!!

That's a lot of files and processing to do transcoding Now we have all the files we need to stream it. OC Open connect comes in to picture, OC is Netflix own CDN no third-party CDN

### **Advantages of OC**
* Less expensive
* Better quality
* More Scalable

So once the videos are transcoded these files are pushed to all of the OC servers.

### **Now the second scenario:**

When the user loads Netflix app All requests are handled by the server in AWS Eg: Login, recommendations, home page, users history, billing, customer support etc. Now you want to watch a video when you click the play button of the Video. Your app automatically figures out the best OC server, best format and best bitrate for you and then the video is streamed from a nearby Open Connect Appliance (OCA) in the Open Connect CDN.

The Netflix apps are so intelligent that they constantly check for best streaming server and bitrate for you and switches between formats and servers to give the best viewing experience for you.
Now what Netflix does is with all of your searches, viewing, location, device, reviews and likes data on AWS it uses Hadoop | Machine learning models to recommend new movies which you might like...
And this cycle goes on and on

Netflix supports 2200 different devices, including Smart TV, Adroid, IOS, gaming consoles, web apps etc
All these apps are written in platform-specific code.

### **Netflix Likes React JS:**
React was influenced by a number of factors, most notably: 1) *startup speed*, 2) *runtime performance*, and 3) *modularity*

### **ELB:**
Netflix uses Amazons Elastic Load Balancer (ELB) service to route traffic to our front-end services. ELB’s are set up such that load is balanced across zones first, then instances. This is because the ELB is a two-tier load balancing scheme.

* The first tier consists of basic DNS based round robin load balancing. This gets a client to an ELB endpoint in the cloud that is in one of the zones that your ELB is configured to use.
* The second tier of the ELB service is an array of load balancer instances (provisioned directly by AWS), which does round-robin load balancing over our own instances that are behind it in the same zone.

### **ZUUL:**
Zuul is the front door for all requests from devices and websites to the backend of the Netflix streaming application. As an edge service application, Zuul is built to enable dynamic routing, monitoring, resiliency, and security. 

When we work with a Gateway Service like Zuul, probably we want to include a Circuit Breaker mechanism to avoid ugly errors in case of redirecting to a service which is unavailable or not responding in time. We can do that by connecting Hystrix with Zuul with a ZuulFallbackProvider bean.
What we want to accomplish here is a better error recovery strategy when a service behind a gateway is failing. In that scenario, the problem is that Zuul would return an Internal Server Error, which might end up crashing a web page or just giving a bad time to our REST API consumers.

We can avoid that using a Circuit Breaker pattern and, with Spring Boot, the best-integrated implementation is Spring Cloud Netflix Hystrix.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/blog_zuul_hystrix.png)

Routing is an integral part of a microservice architecture. For example, /api/users is mapped to the user service and /api/shop is mapped to the shop service. Zuul is a JVM-based router and server side load balancer by Netflix.

The volume and diversity of Netflix API traffic sometimes results in production issues arising quickly and without warning. We need a system that allows us to rapidly change behavior in order to react to these situations.

Zuul uses a range of different types of filters that enables us to quickly and nimbly apply functionality to our edge service. These filters help us perform the following functions: 

*    Authentication and Security: identifying authentication requirements for each resource.
*    Insights and Monitoring: tracking meaningful data and statistics.
*    Dynamic Routing: dynamically routing requests to different backend..
*    Stress Testing: gradually increasing the traffic.
*    Load Shedding: allocating capacity for each type of request and dropping requests.
*    Static Response handling: building some responses directly.
*    Multiregion Resiliency: routing requests across AWS regions.

Zuul contains multiple components:

*    zuul-core: library that contains the core functionality of compiling and executing Filters.
*    zuul-simple-webapp: webapp that shows a simple example of how to build an application with zuul-core.
*    zuul-netflix: library that adds other NetflixOSS components to Zuul — using Ribbon for routing requests, for example.
*    zuul-netflix-webapp: webapp which packages zuul-core and zuul-netflix together into an easy to use package.

The Netty handlers on the front and back of the filters are mainly responsible for handling the network protocol, web server, connection management and proxying work. With those inner workings abstracted away, the filters do all of the heavy lifting.

#### How Zuul 2 Works
![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/how_zuul2_works.png)

The **inbound filters** run before proxying the request and can be used for authentication, routing, or decorating the request.
The **endpoint filters** can either be used to return a static response or proxy the request to the backend service (or origin as we call it).
The **outbound filters** run after a response has been returned and can be used for things like gzipping, metrics, or adding/removing custom headers.

Zuul’s functionality depends almost entirely on the logic that you add in each filter. That means you can deploy it in multiple contexts and have it solve different problems based on the configurations and filters it is running.
Netflix uses Zuul at the entrypoint of all external traffic into Netflix’s cloud services and we’ve started using it for routing internal traffic, as well. We deploy the same core but with a substantially reduced amount of functionality (i.e. fewer filters). This allows us to leverage load balancing, self service routing, and resiliency features for internal traffic.


#### **Features:**
*	Supports http2
*	mutual TLS
*	Adaptive retries
*	Concurrency protection for the origin

It helps in Easy routing based on query params, URL, path. The main use case is for routing traffic to a specific test or staging cluster.

#### What is the Netflix Zuul? Need for it?
Zuul is a JVM based router and server side load balancer by Netflix. 
It provides a single entry to our system, which allows a browser, mobile app, or other user interface to consume services from multiple hosts without managing cross-origin resource sharing (CORS) and authentication for each one. We can integrate Zuul with other Netflix projects like Hystrix for fault tolerance and Eureka for service discovery, or use it to manage routing rules, filters, and load balancing across your system.

* Microservice call without Netflix Zuul

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/without_zuul.jpg)

* Microservice call with Netflix Zuul

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/with_zuul.jpg)

* Microservice call with Netflix Zuul + Netflix Eureka 

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/with_zuul_eureka.jpg)

Refer https://www.javainuse.com/spring/spring-cloud-netflix-zuul-tutorial for more details...

**Zuul and NGINX are very similar the key differences are**

Unlike for NGINX Zuul can’t be downloaded as a binary distribution, instead it is running inside a Spring Boot application using the Spring Cloud integration
Zuul is written in Java, therefore integration with the Java, Spring Boot, Spring Cloud and Netflix OSS ecosystem is quite easy

#### Advantages: ??
* Services needing to shard their traffic create routing rules that map certain paths or prefixes to separate origins
* Developers onboard new services by creating a route that maps a new hostname to their new origin
* Developers run load tests by routing a percentage of existing traffic to a small cluster and ensuring applications will degrade gracefully under load
* Teams refactoring applications migrate to a new origin slowly by creating rules mapping traffic gradually, one path at a time
* Teams test changes (canary testing) by sending a small percentage of traffic to an instrumented cluster running the new build
* If teams need to test changes requiring multiple consecutive requests on their new build, they run sticky canary tests that route the same users to their new build for brief periods of time
* Security teams create rules that reject "bad" requests based on path or header rules across all Zuul clusters

#### Hystrix:
Hystrix is a latency and fault tolerance library designed to isolate points of access to remote systems, services and 3rd party libraries. which helps in

* Stop cascading failures
* Real-time monitoring of configurations changes
* Concurrency aware request caching
* Automated batching through request collapsing

IE. If a microservice is failing then return the default response and wait until it recovers.

### Microservices:
**How APIS gets the response?**

Netflix uses MicroServices architecture to power all of the APIs needed for applications and Web apps. Each API calls the other micro-services for required data and then responds with the complete response

*This setup works, but is it reliable?*

* We can use Hysterix which I already explained
* We separate critical services

### Critical Microservices:
What Netflix does is, they identify few services as critical (so that at last user can see recommended hit and play, in case of cascaded service failure) and these micro-services works without many dependencies to other services !!!

### Stateless Services:
One of the major design goals of the Netflix architecture’s is stateless services.

These services are designed such that any service instance can serve any request in a timely fashion and so if a server fails it’s not a big deal. In the failure, case requests can be routed to another service instance and we can automatically spin up a new node to replace it.

### EVCache:
When a node goes down all the cache goes down along with it. so the performance hit until all the data is cached. so what Netflix did is they came up with EVcache. It is a wrapper around Memcached but it is sharded so multiple copies of cache is stored in sharded nodes. So every time the write happens, all the shards are updated too...

When cache reads happens, read from nearest cache or nodes, but when a node is not available, read from a different available node. It handles 30 million request a day and linear scalability with milliseconds latency.

### SSDs for Caching:
Storing large amounts of data in volatile memory (RAM) is expensive. Modern disk technologies based on SSD are providing fast access to data but at a much lower cost when compared to RAM. Hence, we wanted to move part of the data out of memory without sacrificing availability or performance. The cost to store 1 TB of data on SSD is much lower than storing the same amount of RAM.

### Database: (EC2 deployed MySQL)
EC2 MySQL was ultimately the choice for the billing/user info use case, Netflix built MySQL using the InnoDB engine large ec2 instances. They even had master-master like setup with “Synchronous replication protocol” was used to enable the write operations on the primary node to be considered completed. Only after both the local and remote writes have been confirmed.

As a result, the loss of a single node is guaranteed to have no data loss. This would impact the write latency, but that was well within the SLAs.

Read replica set up in local, as well as cross-region, not only met high availability requirements, but also helped with scalability.

The read traffic from ETL jobs was diverted to the read replica, sparing the primary database from heavy ETL batch processing. In case of the primary MySQL database failure, a failover is performed to the secondary node that was being replicated in synchronous mode. Once secondary node takes over the primary role, the **route53** DNS entry for database host is changed to point to the new primary.

**Cassandra: -> 500 nodes 50 clusters**
Cassandra is a free and open-source distributed wide column store **NoSQL** database designed to handle large amounts of data across many commodity servers, providing high availability with no single point of failure

At Netflix as userbase started to grow more there has been a massive increase in viewing history data.

Initial days it was fine, but not for long. So Netflix Redesigned data storage arch with two main goals in mind:

* Smaller Storage Footprint.
* Consistent Read/Write Performance as viewing per member grows.

So the solution: Compress the old rows!! Data was divided in to two types

**Live Viewing History (LiveVH):** Small number of recent viewing records with frequent updates. The data is stored in uncompressed form as in the simple design detailed above.

**Compressed Viewing History (CompressedVH):** Large number of older viewing records with rare updates. The data is compressed to reduce storage footprint. Compressed viewing history is stored in a single column per row key.

**Kafka to chukwa for distribute system monitoring**

Push all the netflix events to processing pipelines

~500 billion events and ~1.3 PB per day

~8 million events and ~24 GB per second during peak hours

*What kind of events??*

* Video viewing activities
* UI activities
* Error logs
* Performance events
* Troubleshooting & diagnostic events

**Apache Chukwa** is an open source data collection system for monitoring large distributed systems. Apache Chukwa is built on top of the Hadoop Distributed File System (HDFS) and Map/Reduce framework and inherits Hadoop’s scalability and robustness.

Apache Chukwa also includes a ﬂexible and powerful toolkit for displaying, monitoring and analyzing results to make the best use of the collected data.

The kafka routing service is responsible for moving data from fronting Kafka to various sinks: *S3, Elasticsearch, and secondary Kafka*.

**Routing is done using apache Samza**

When Chukwa sends traffic to Kafka, it can deliver full or filtered streams. Sometimes, we need to apply further filtering on the Kafka streams written from Chukwa. That is why we have the router to consume from one Kafka topic and produce to a different Kafka topic.

**Elastic search:**
We have seen explosive growth in Elastic search adoption within Netflix for the last two years. There are ~150 clusters totaling ~3,500 instances hosting ~1.3 PB of data. The vast majority of the data is injected via our data pipeline.

*How its used at Netflix*
Say when a customer tried to play a video and he couldn’t, he calls the customer care now how customer care guys can debug whats happening? So Playback team uses Elastic search to drill down the problem and also to understand how widespread is the problem.

* To see Signup or login problems
* To keep track of resource usage

**AWS Application Auto Scaling feature — TITUS**
Titus is a container management platform that provides scalable and reliable container execution and cloud-native integration with Amazon AWS.

Titus was built internally at Netflix and is used in production to power Netflix streaming, recommendation, and content systems.

It also has scheduling support for service applications. Mainly used to scale docker images, It talks to AWS auto scale service using AWS API gateways to scale dockers on AWS.

AWS auto scale can scale instances, Titus will scale instances and also dockers based on the traffic conditions. As Netflix has many micro services on docker.

For example, as people on the east coast of the U.S. return home from work and turn on Netflix, services automatically scale up to meet this demand. Scaling dynamically with demand rather than static sizing helps ensure that services can automatically meet a variety of traffic patterns without service owners needing to size and plan their desired capacity. Additionally, dynamic scaling enables cloud resources that are not needed to be used for other purposes, such as encoding new content.

This design centered around the AWS Auto Scaling engine being able to compute the desired capacity for a Titus service, relay that capacity information to Titus, and for Titus to adjust capacity by launching new or terminating existing containers. There were several advantages to this approach. First, Titus was able to leverage the same proven auto scaling engine that powers AWS rather than having to build our own. Second, Titus users would get to use the same Target Tracking and Step Scaling policies that they were familiar with from EC2. Third, applications would be able to scale on both their own metrics, such as request per second or container CPU utilization, by publishing them to CloudWatch as well as AWS-specific metrics, such as SQS queue depth. Fourth, Titus users would benefit from the new auto scaling features and improvements that AWS introduces.

The key challenge was enabling the AWS Auto Scaling engine to call the Titus control plane running in Netflix’s AWS accounts. To address this, we leveraged AWS API Gateway, a service which provides an accessible API “front door” that AWS can call and a backend that could call Titus. API Gateway exposes a common API for AWS to use to adjust resource capacity and get capacity status while allowing for pluggable backend implementations of the resources being scaled, such as services on Titus. When an auto scaling policy is configured on a Titus service, Titus creates a new scalable target with the AWS Auto Scaling engine.

**Media processing while onboarding and later**
Validating the video: The first thing Netflix does is spend a lot of time validating the video. It looks for digital artifacts, color changes, or missing frames that may have been caused by previous transcoding attempts or data transmission problems.

The video is rejected if any problems are found.

After the video is validated, it’s fed into what Netflix calls the media pipeline.

A pipeline is simply a series of steps data is put through to make it ready for use, much like an assembly line in a factory. More than 70 different pieces of software have a hand in creating every video.

It’s not practical to process a single multi-terabyte sized file, so the first step of the pipeline is to break the video into lots of smaller chunks.

The video chunks are then put through the pipeline so they can be encoded in parallel. In parallel simply means the chunks are processed at the same time

### Archer:
Archer is an easy to use MapReduce style platform for media processing that uses containers so that users can bring their OS-level dependencies. Common media processing steps such as mounting video frames are handled by the platform. Developers write three functions: split, map and collect; and they can use any programming language. Archer is explicitly built for simple media processing at scale, and this means the platform is aware of media formats and gives white glove treatment for popular media formats.

For example, a PRORES video frame is a first class object in Archer and splitting a video source into shot based chunks [1] is supported out of the box (a shot is a fragment of the video where the camera doesn’t move).

1. detecting dead pixels caused by defective digital camera
2. machine learning (ML) to tag audio
3. QC for subtitles

### SPARK Usage for the movierecommendation
Spark is used for content recommendations and personalization. A majority of the machine learning pipelines for member personalization run atop large managed Spark clusters. These models form the basis of the recommender system that backs the various personalized canvases you see on the Netflix app including, title relevance ranking, row selection & sorting, and artwork personalization among others.

### Netflix personalizes artwork just for you
Here’s a great example of how Netflix entices you to watch more videos using its data analytics capabilities.

When browsing around looking for something to watch on Netflix, have you noticed there’s always an image displayed for each video? That’s called the header image.

The header image is meant to intrigue you, to draw you into selecting a video. The idea is the more compelling the header image, the more likely you are to watch a video. And the more videos you watch, the less likely you are to unsubscribe from Netflix.

Here’s an example of different header images for Stranger Things:

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/netflix_header.jpeg)

You might be surprised to learn the image shown for each video is selected specifically for you. Not everyone sees the same image.

Everyone used to see the same header image. Here’s how it worked. Members were shown at a random one picture from a group of options, like the pictures in the above Stranger Things collage. Netflix counted every time the video was watched, recording which picture was displayed when the video was selected.

For our Stranger Things example, let’s say when the group picture in the center was shown, Stranger Things was watched 1,000 times. For all the other pictures, it was watched only once each.

Since the group picture was the best at getting members to watch, Netflix would make it the header image for Stranger Things forever.

This is called being data-driven. Netflix is known for being a data-driven company. Data is gathered — in this case, the number of views associated with each picture — and used to make the best decisions possible — in this case, which header image to select.

Clever, but can you imagine doing better? Yes, by using more data. That’s the theme of the future — solving problems by learning from data.

You and I are likely very different people. Do you think we are motivated by the same kind of header image? Probably not. We have different tastes. We have different preferences.

Netflix knows this too. That’s why Netflix now personalizes all the images they show you. Netflix tries to select the artwork highlighting the most relevant aspect of a video to you. How do they do that?

Remember, Netflix records and counts everything you do on their site. They know which kind of movies you like best, which actors you like the most, and so on.

### How Netflix’s Recommendations System Works
Read the urls below and delete once done...
http://blog.gainlo.co/index.php/2016/05/24/design-a-recommendation-system/
https://www.datacamp.com/community/tutorials/recommender-systems-python?utm_source=adwords_ppc&utm_campaignid=1455363063&utm_adgroupid=65083631748&utm_device=c&utm_keyword=&utm_matchtype=b&utm_network=g&utm_adpostion=&utm_creative=332602034361&utm_targetid=aud-299261629574:dsa-473406569915&utm_loc_interest_ms=&utm_loc_physical_ms=9040195&gclid=Cj0KCQiA88X_BRDUARIsACVMYD8yUYPPIGYoxY66_EJVKTspOgLUe59R3YoZpShpblKqHoj4OUBuTiQaAu2fEALw_wcB
file:///Users/samirsahoo/Downloads/Design_of_a_recommendation_system_based_on_collabo.pdf
https://realpython.com/build-recommendation-engine-collaborative-filtering/
https://uxplanet.org/how-can-we-design-an-intelligent-recommendation-engine-b9bb1db4d050

https://beta.vu.nl/nl/Images/werkstuk-fernandez_tcm235-874624.pdf

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/recommendation.png)

Whenever you access the Netflix service, our recommendations system strives to help you find a show or movie to enjoy with minimal effort. We estimate the likelihood that you will watch a particular title in our catalog based on a number of factors including:

* your interactions with our service (such as your viewing history and how you rated other titles),
* other members with similar tastes and preferences on our service (more info here), and
* information about the titles, such as their genre, categories, actors, release year, etc.

In addition to knowing what you have watched on Netflix, to best personalize the recommendations we also look at things like:

* the time of day you watch,
* the devices you are watching Netflix on, and
* how long you watch.

**Collaborative filtering**(CF) algorithms are based on the idea that if two clients have similar rating history then they will behave similarly in the future (Breese, Heckerman, and Kadie, 1998). If, for example, there are two very likely users and one of them watches a movie and rates it with a good score, then it is a good indication that the second user will have a similar pattern

**Content-based filtering**(CB) aims to recommend items or movies that are alike to movies the user has liked before. The main difference between this approach and the CF is that CB offers the recommendation based not only in similarity by rating, but it is more about the information from the products (Aggarwal, 2016), i.e., the movie title, the year, the actors, the genre. In order to implement this methodology, it is necessary to possess information describing each item, and some sort of user profile describing what the user likes is also desirable. The task is to learn the user preferences, and then locate or recommend items that are “similar” to the user preferences

**Hybrid filtering** methods are characterized by combining CF and CB techniques


## API DESIGN

In today’s world, a lot of systems support mobile platform so APIs are the best choices to be able to provide the distinction between developers and support mobile support as well. We can use REST or SOAP. A lot of huge companies prefer to REST or SOAP according to their systems. There are three main API’s we will mention below:

   1- UploadVideo(apiKey, title, description, categoryID, language)

     Upload video is the first API that we should mention. There are basically five main properties of this API. You can add more properties to UploadVideo API. Note that, apiKey is the developer key of registered account of service. Thanks to apiKey we can eliminate hacker attacks. UploadVideo returns the HTTP response that demonstrates video is uploaded successfully or not.

   2- DeleteVideo (apiKey, videoID)

     Check if user has permission to delete video. It will return HTTP response 200 (OK), 202 (Accepted) if the action has been queued, or 204 (No Content) based on your response.

   3- GetVideo (apiKey, query, videoCountToReturn, pageNumber)

     Return JSON containing information about the list of videos and channels. Each video resource will have a title, creation date, like count, total view count, owner and other meta informations.

**There are more APIs to design video sharing service, however, these three APIs are more important than the others. Other APIs will be like likeVideo, addComment, search, recommendation or etc...**

## DATABASE DESIGN

There are two choices to define the database schema. These are SQL and NoSQL. We can use traditional database management system like MsSQL or MySQL to keep data. As you know, we should keep information about videos and users into RDBMS.  Other information about videos, called metadata, should be kept too. Now we have the main three tables to keep data. (Notice that we just only think the basic properties of Youtube. We can forget the recommendation system).

  User

    – UserID (primary key)
    – Name (nvarchar)
    – Age (Integer)
    – Email (nvarchar)
    – Address (nvarchar)
    – Register Date (DateTime)
    – Last Login (DateTime)



  Video

    – VideoID (primary key – generated by KGS – Key generation service)
    – VideoTitle (nvarchar)
    – Size (float)
    – UserID (foreign key with User Table)
    – Description (nvarchar)
    – CategoryID (int) : Note that we can create Category Table to define categories
    – Number of Likes (int)
    – Number of Dislikes (int)
    – Number of Displayed (int) – We can use big int to keep displayed number
    – Uploaded Date (DateT?me)

  VideoComment

    – CommentID (primary key)
    – UserID (foreign key with User Table)
    – VideoID (foreing key with Video Table)
    – Comment (nvarchar)
    – CommentDate (DateTime)


### BASIC CODING OF SYSTEM
```
// Java program for design 
public enum AccountStatus{ 
  PUBLIC, 
  PRIVATE, 
  CLOSED 
} 
  
public enum VideoContentStatus { 
  PENDING, 
  PROCESSED, 
  FAIL, 
  REJECTED 
} 
  
public enum VideoStatus { 
  PUBLIC, 
  PRIVATE, 
  DELETED 
} 
  
public enum VideoQuality { 
  LOW, 
  MIDDLE, 
  HIGH 
} 
  
public class AddressDetails { 
  private String street; 
  private String city; 
  private String country; 
  ... 
} 
  
public class AccountDetails { 
  private Date createdTime; 
  private AccountStatus status; 
  private boolean updateAccountStatus(AccountStatus accountStatus); 
  ... 
} 
  
public class Comment { 
  private Integer id; 
  private User addedBy; 
  private Date addedDate; 
  private String comment; 
  
  public boolean updateComment(String comment); 
  ... 
} 
  
public class Video { 
  private Integer id; 
  private User createdBy; 
  private String path; 
  private VideoStatus videoStatus; 
  private VideoContentStatus videoContentStatus; 
  private int viewsCount; 
  
  private HashSet<Integer> userLikes; 
  private HashSet<Integer> userDisLikes; 
  private HashSet<Integer> userComments; 
  ... 
} 
  
public class User { 
  private int id; 
  private String password; 
  private String nickname; 
  private String email; 
  private AddressDetails addressDetails; 
  private AccountDetails accountDetails; 
  private UserRelations userRelations; 
  private HashSet<ConnectionInvitation> invitationsByMe; 
  private HashSet<ConnectionInvitation> invitationsToMe; 
  
  private boolean updatePassword(); 
  public boolean uploadVideo(Video video); 
  public List<Videos> getVideos(); 
  ... 
}
```
Ref:
https://medium.com/@narengowda/netflix-system-design-dbec30fede8d
https://www.geeksforgeeks.org/design-video-sharing-system-like-youtube/


# Spring Cloud for Microservices Compared to Kubernetes

Spring Cloud and Kubernetes both claim to be the best environment for developing and running Microservices, but they are both very different in nature and address different concerns. 
The two platforms, Spring Cloud and Kubernetes, are very different and there is no direct feature parity between them. If we map each MSA concern to the technology/project used to address it in both platforms, we come up with the following table.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/spring_cloud_vs_kubernetes.png)

The main takeaways from the above table are:
* Spring Cloud has a rich set of well integrated Java libraries to address all runtime concerns as part of the application stack. As a result, the Microservices themselves have libraries and runtime agents to do client side service discovery, load balancing, configuration update, metrics tracking, etc. Patterns such as singleton clustered services and batch jobs are managed in the JVM too.
* Kubernetes is polyglot, doesn't target only the Java platform, and addresses the distributed computing challenges in a generic way for all languages. It provides services for configuration management, service discovery, load balancing, tracing, metrics, singletons, scheduled jobs on the platform level and outside of the application stack. The application doesn't need any library or agents for client side logic and it can be written in any language.
* In some areas both platforms rely on similar third party tools. For example the ELK and EFK stacks, tracing libraries, etc. Some libraries such as Hystrix, Spring Boot are useful equally well on both environments. There are areas where both platforms are complementary and can be combined together to create a more powerful solution (KubeFlix and Spring Cloud Kubernetes are such examples).

Ref: http://www.ofbizian.com/2016/12/spring-cloud-compared-kubernetes.html

Read the article and setup here ![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/Infrastructure_services.png)

A better design...

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/system-landscape.png)
Refer: https://callistaenterprise.se/blogg/teknik/2015/04/15/building-microservices-with-spring-cloud-and-netflix-oss-part-2/


If you use Ribbon and Eureka in your Spring Boot application, you’ll notice that the default configuration is not optimal. Eureka takes a long time to figure out that the service went down unexpectedly and, in the meantime, your load balancer Ribbon will try to connect to the dead one. In other words, the eviction policy does not perform well by default.

On the other hand, the official Eureka documentation discourages changing the leaseRenewalIntervalInSeconds parameter so, what can we do here?

## Netflix Eureka does not deregister instances

Eureka is a great tool for Service Discovery and integrates very well with Spring Boot. We can create a Service Registry just by adding some dependencies and an annotation, and we can connect our clients to register on the server with minimal configuration too. It comes with Ribbon as Load Balancer, so we can get everything working in few minutes. Our services will ask the registry for the instances of a given service and will decide by themselves, using Ribbon, to which one they’re connecting.

The problem arises when one of our multiple instances of a service goes down unexpectedly or loses connectivity, not having time to notify the Eureka server. The service registry is based on leases, and every client should renew itself every N seconds. When the lease expires, Eureka takes also some time to decide that the instance is no longer valid. It’s not a very straightforward mechanism, as you’ll see explained on different Internet threads. With the default configuration, my experience is that it can take up to four or five minutes to deregister a dead instance.

On the other hand, the official documentation tells us that we shouldn’t change this configuration. My guess is that, since the mechanism to deregister is not easy to understand, you can mess the entire configuration up if you make mistakes.

## How to solve it

The approach to solving this problem is based on Ribbon and not Eureka. We can leave the Service Registry alone and let it work as usual, with the optimal configuration, but we can make our clients smarter and let them find out which services are really healthy.

Spring Boot configures Ribbon by default with a Round-Robin strategy for load balancing. It also sets the status-check mechanism to none (NoOpPing), which means that the load balancer will not verify if the services are still alive. It makes sense since it should be our Service Registry, Eureka, the one that registers and deregisters instances. But we just concluded that the time it will take it’s not good for us.

## Adding Ribbon Configuration

We can use the Ribbon functionality to ping services from the Service Registry and apply load balancing depending on the result. To get that working, we need to configure two Spring beans: an IPing to establish the check-status mechanism and an IRule to change the default load balancing strategy.

* The PingUrl implementation checks if services are alive. We want to change the default URL and point it to /health since we want to avoid requests to unmapped root contexts. The false flag is just to indicate that the endpoint is not secured.
* The AvailabilityFilteringRule is an alternative to the default RoundRobinRule that takes into account the availability being checked by our new pings.
* One thing that's very important to note (since it's tricky) is that this class is not annotated with @Configuration. It's injected in a different way: we need to reference it from a new annotation added to the main application class: @RibbonClients.

If we test now again the scenario where multiple instances are registered, and then one of them goes down, we’ll notice that the reaction time to find out an unavailable service is much less.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/logical_view_v8-1024x977.png)

Ref: https://thepracticaldeveloper.com/how-to-fix-eureka-taking-too-long-to-deregister-instances/



* **Eureka** is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the purpose of load balancing and failover of middle-tier servers. We call this service, the Eureka Server. Eureka also comes with a Java-based client component, the Eureka Client, which makes interactions with the service much easier. The client also has a built-in load balancer that does basic round-robin load balancing. At Netflix, a much more sophisticated load balancer wraps Eureka to provide weighted load balancing based on several factors like traffic, resource usage, error conditions etc to provide superior resiliency.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/microservice_registration.jpg)

* **Spring Cloud Config** provides server-side and client-side support for externalized configuration in a distributed system. With the Config Server, you have a central place to manage external properties for applications across all environments.

* **Zuul** is an edge service that provides dynamic routing, monitoring, resiliency, security, and more.

Ref for config: https://engineering.pivotal.io/post/local-eureka-zuul-cloud_config-with-spring/

# Architecture that common microservices use

* Client sends in a request which is sent to the api gateway.
* The Gateway may utilize a service discovery service to tell the port number of the named service to which the request points to.
* The request is then forwarded to the specified service and the responce is generated from the service.


1. **API Gateway**-

* Netflix Zuul acts as an API gateway
* When we have multiple instances of microservices registered on service discovery server, and we need to hide our system complexity to the outside world, we deploy a API gateway such as Netflix Zuul.
* There should be only one IP address exposed on one port available for inbound clients. That’s why we need API gateway — Zuul.
* Zuul will forward our request to the specific microservice based on its proxy configuration.
* Zuul is an edge service that provides dynamic routing, monitoring, resiliency, security, and more. Please view the wiki for usage, information, HOWTO, etc https://github.com/Netflix/zuul/wiki

2. **Discovery Service**-

* In order for a client to make a request to a service it must use a service-discovery mechanism. A key part of service discovery is the service registry. The service registry is a database of available service instances.
* This project uses Netflix Eureka for service discovery: Eureka is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the purpose of load balancing and failover of middle-tier servers. For more info on Eureka read on : https://github.com/Netflix/eureka/wiki/Eureka-at-a-glance

3. **Ribbon Client**-

* Ribbon Load Balancer is integrated with Netflix OSS and is used for load balancing in the microservice architecture. Ribbon is a client side IPC(Inter-Process-Communication) library that is battle-tested in cloud.

4. **FEIGN Client**-

* Feign makes writing java http clients easy and simple. It is utilized for inter-process communication in the project. To read more on Feign go to: https://github.com/OpenFeign/feign

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/microservices_zuul_eureka.png)

## Request Flow

* When we hit the url : http://localhost:8765/api/customer/customers (after starting the required services) (discovery-service; gateway-service; customer-service; account-service; document-service; zipkin)
* The Client request is recieved at the Gateway API.
* The Gateway API(Zuul) resolves the port number from the service name using the Discovery Service (Eureka)
* The request is sent to the customer microservice’s (findall()) method which internally uses the feign Client to take data from the Account MicroService.
* The Account Microservice inturn hits the Document MicroService’s (findall()) method using the feign Client
* The data is loaded from Document MicroService to Account MicroService and from the Account MicroService to the Customer microService.
* The complete data is then sent as a responce back to the client.





# One: What is Microservice?
 
Microservice architecture mode is to organize the entire web application into a series of small Web services. These small Web services can be compiled and deployed independently and communicate with each other through their exposed API interfaces. They work together to provide functionality to the user as a whole, but can be extended independently.

Features or usage scenarios required by the microservices architecture

* 1: We split the entire system into several subsystems based on the business.

* 2: Multiple applications can be deployed per subsystem, and load balancing is used between multiple applications.

* 3: A service registry is required. All services are registered in the registry. Load balancing is also achieved by using certain policies through the services registered in the registry.

* 4: All clients access the background service through the same gateway address. Through routing configuration, the gateway determines which service a URL request is processed by. Load balancing is also used when requests are forwarded to the service.

* 5: Sometimes services need to be accessed from each other. For example, there is a user module, and other services need to obtain user data of the user service when processing some services.

* 6: A circuit breaker is needed to deal with timeouts and errors in the service call in time to prevent the overall system from being paralyzed due to problems with one of the services.

* 7: A monitoring function is also needed to monitor the time spent on each service call.

At present, the mainstream micro-service framework: Dubbo, SpringCloud, thrift, Hessian, etc., most of the domestic SMEs use Dubbo, SpringCloud is estimated to be rare, and perhaps some development students have not heard of it.

# Two: Introduction to the SpringCloud project

springCloud is a set of frameworks for implementing microservices based on SpringBoot. He provides components such as configuration management, service discovery, circuit breakers, intelligent routing, micro-proxy, control bus, global locks, decision-making campaigns, distributed sessions, and cluster state management required for microservice development. the most important is,

When used with the spring boot framework, it will make it very convenient for you to develop a cloud service for the microservice architecture.

SpringBoot is designed to simplify the creation of product-level Spring applications and services, simplify configuration files, use embedded web servers, and includes many out-of-the-box microservices

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/spring_cloud.jpg) 

The spring cloud subproject includes:

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/eureka.jpg)

**Spring Cloud Config**: The Configuration Management Development Kit allows you to put your configuration on a remote server and currently supports local storage, Git, and Subversion.

**Spring Cloud Bus**: An event, message bus that propagates state changes in a cluster (for example, configuration change events) and can be combined with Spring Cloud Config for hot deployment.

Spring Cloud Netflix: A development kit for a variety of Netflix components, including Eureka, Hystrix, Zuul, Archaius, and more.

Netflix Eureka: Cloud Load Balancing, a REST-based service for locating services for cloud-based load balancing and middle-tier server failover.

Netflix Hystrix: A fault-tolerant management tool designed to provide greater fault tolerance for latency and failure by controlling the nodes of services and third-party libraries.

Netflix Zuul: The Edge Services tool is an edge service that provides dynamic routing, monitoring, resiliency, security, and more.

Netflix Archaius: A configuration management API that includes a set of configuration management APIs that provide dynamic typing properties, thread-safe configuration operations, polling frameworks, callback mechanisms, and more.

Spring Cloud for Cloud Foundry: Binding services to CloudFoundry via Oauth2 protocol, CloudFoundry is an open source PaaS cloud platform from VMware.

Spring Cloud Sleuth: A log collection toolkit that encapsulates Dapper, Zipkin and HTrace operations.

Spring Cloud Data Flow: A big data manipulation tool that manipulates data streams from the command line.

Spring Cloud Security: A security toolkit that adds security controls to your application, primarily OAuth2.

Spring Cloud Consul: Encapsulates the Consul operation, a service discovery and configuration tool that seamlessly integrates with Docker containers.

Spring Cloud Zookeeper: A toolkit for operating Zookeeper for service registration and discovery using the zookeeper method.

Spring Cloud Stream: Data flow operation development package, which encapsulates sending and receiving messages with Redis, Rabbit, Kafka, etc.

Spring Cloud CLI: Based on the Spring Boot CLI, you can quickly build cloud components from the command line.

Turbine — Has the ability to get the metrics from different services and provide a stream combining them.

Feign — provides a way to wrap rest call from one service to another service in a nice spring-ish manner.
  

SpringCloud features

* 1: Convention is better than configuration

* 2: Out of the box, quick start

* 3: Suitable for various environments

* 4: Lightweight components

* 5: rich component support, full-featured


# Three: SpringBoot understand
 

Spring Boot makes our Spring applications lighter. For example: you can rely on a Java class to run a Spring reference. You can also package your app as a jar and run your Spring web app by using java -jar.

Since SpringCloud relies on SpringBoot, you need to understand SpringBoot before learning the SpringCloud framework.

The main advantages of SpringBoot:

1: Get started faster for all Spring developers

2: Out of the box, with various default configurations to simplify project configuration

3: Embedded container simplifies web projects

4: No requirements for redundant code generation and XML configuration

Ref: https://www.programmersought.com/article/82911571660/
 
# Anomaly Detection and Contextual Alerting

As we grew from just a handful of origins to a new world where anyone can quickly spin up a container cluster and put it behind Zuul, we found there was a need to automatically detect and pinpoint origin failures.

With the help of Mantis real time event streaming, we built an anomaly detector that aggregates error rates per service and notifies us in real time when services are in trouble. It takes all of the anomalies in a given time window and creates a timeline of all the origins in trouble. We then create a contextual alert email with the timeline of events and services affected. This allows an operator to quickly correlate these events and orient themselves to debug a specific app or feature, and ultimately find the root cause.

**Notes**:

Netflix uses Zuul2 and has opensourced it.

 The Cloud Gateway team at Netflix runs and operates more than 80 clusters of Zuul 2, sending traffic to about 100 (and growing) backend service clusters which amounts to more than 1 million requests per second. 


# Build a sample project with spring cloud using cloud config, eureka, zuul, feign, hystrix and turbine

Sample ready to use project with all basic configuration for -
```
1.    Spring cloud config (taking out and putting all the properties file at the same location)
2.    Eureka discovery enabled to maintain scaled up instances
3.    API gateways with zuul to support routing to different microservice
4.    Use of feign client to call different micro service wherever necessary
5.    fallback support for feign
6.    hystrix dashboard with turbine to get metrics
```
https://github.com/samirsahoo007/microservices/tree/master/cloud-pet-project

## The use case story

We need to have a system where we can get the pet info, for starter we would need info about two types of pet cats and dogs

* The ability to have different service for cats and dogs info, which would help me in scaling the service separately.
* However the pet food preference info should be a separate service, which provides the info for all kinds of pets.
* We need a fallback as if the info from food service isn’t available then we show custom message.
* There should be a single point of contact, which means I should be able to call a single interface for cats and dogs both however the interface communicate with respective services.
* Should have the ability to visualize the load and interactions on different services on a common platform.

Look at the **Two: Introduction to the SpringCloud project** above for all the Tools.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/cloud_pet_project.png)

## Understanding the components

### Config server
To start the project, first start config server application
```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

After that we need to annotate the application to be run as a cenralized config server with @EnableConfigServer
If it is set up to be used in native mode, it will search for the properties file specifically in local folder otherwise through a git repo.
local folder option is good for local testing however any other environment requires a separate git repo of all your properties file
```
Local option : -
spring.cloud.config.server.native.search-locations: file:\\\${HOME}\Desktop\pet_profiles_config
Git option :-
spring.cloud.config.server.git.uri=https://my.git.profile.location.git
spring.cloud.config.server.git.username=<username>
spring.cloud.config.server.git.password=<password>
spring.cloud.config.server.git.label=master # optionally the branch name
```

### Eureka server
Eureka server provides a common platform for all the microservices to communicate with each other or to be more correct get info about each other
```
<dependency>
 <groupId>org.springframework.cloud</groupId>
 <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId></dependency>
```

After that we need to annotate the application to be run as a eureka server with @EnableEurekaServer
the bootstrap.properties to fetch the config from pet-config should be as below

Once the service is running you should see the registered server on http://localhost:8761

#### Cat service
Cat service is the service which holds the info for the cats through APIs, it needs to be discovered by eureka server, should get properties from config server, should make a call to food service when required, should also provide fallback through circuit breaker
```
<dependency>
  <groupId>org.springframework.cloud</groupId>
 <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
It needs few self explanatory annotations too as

To use feign client as bean, we need an interface which works like a template

The value in @FeignClient is basically the name of food service provided as spring.application.name
We have also defined a fallback as below

#### Dog service
Everything same as cat service just with different APIs

#####Pet food
It’s more simplistic service and it just needs @EnableDiscoveryClient and target jar spring-cloud-starter-netflix-eureka-client

#####Pet api
Here We would need zuul to route our call to target service
<dependency>
  <groupId>org.springframework.cloud</groupId>
 <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId></dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>

so what the properties for zuul routing looks like?

strip-prefix is by default true, what it does is while routing it removes the url prefix of /cat or /dog , in this case we want to keep it so we are setting it to false.
#####Pet Dashboard
Here we need to enable dashboard as well as turbine stream
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
</dependency>
Here also we would need to enable the annotations as below

##### Flow
UI/web --> pet-api --zuul--> cat and dog --feign--> pet-food
		|_____________|_______|________________|
				   |
				pet-eureka

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/flow.png)

##### Zuul
API gateway is the interface for web servers or browser. API gateway is running on 8080 and through API gateway all other services can be called.
API gateway uses zuul (@EnableZuulProxy) to route the calls to designated microserver
for example : GET http://localhost:8080/app/cat/sound gets routed to GET http://localhost:8070/cat/sound
hitting the API GET http://localhost:8080/app/cat/sound will give us "meooow!" response
  
 similarly hitting API GET http://localhost:8080/app/dog/sound will be routed to dog service and will return "woof!"
##### Feign
Another caveat is when one micro service need to call to other micro service, it will use feign client (@EnableFeignClients). It can be demonstrated as when we call the api gateway to get cat food it routes the call to cat service which in turns makes a call to pet-food service to get cat food information.
GET http://localhost:8080/app/cat/food gets routed to GET http://localhost:8070/cat/food which makes a call to pet-food API http://localhost:8011/food/cat to get the information
##### Circuit breaker and fallback
If for some reason pet-food APIs are unavailable or it cannot be reached, we can configure a fallback implementation by providing @EnableCircuitBreaker and feign.hystrix.enabled=true
To test it, we can take down the pet-food service and try to make the call to http://localhost:8080/app/cat/food, it should show me the below message
food details is not available for cats at the moment! try again later.
##### Hystrix dashboard with turbine
We can keep a track of calls made by routing and circuit breaks by the means of hystrix dashboard, for that reason we would enable turbine (@EnableTurbine) in pet-dashboard running in localhost://8087 and @EnableHystrix in target services
To get the dashboard (@EanbleHystrixDashboard) make a call to http://localhost:8087/hystrix/
You should see the dashboard, in the stream put the value http://localhost:8087/turbine.stream and submit that, it should start monitoring and you should watch different services wuth chart and it should show the real time data when you try to make api calls of cat and dog.
A particular service metrics can be fetched by the call to the service for ex. — http://localhost:8080/actuator/hystrix.stream
##### APIs
 GET  /app/cat/sound
 GET  /app/cat/food
 GET  /app/dog/sound
 GET  /app/dog/food


Ref https://medium.com/@27.rahul.k/build-a-sample-project-with-spring-cloud-using-cloud-config-eureka-zuul-feign-hystrix-and-378b16bcb7c3 for details

# Getting Zuul and Hystrix to work:

The routing configuration, as usual, is located in our application.yml:
```
server:
  port: 8000

zuul:
  ignoredServices: '*'
  prefix: /api
  routes:
    multiplications:
      path: /multiplications/**
      serviceId: multiplication
      strip-prefix: false
    results:
      path: /results/**
      serviceId: multiplication
      strip-prefix: false
    leaders:
      path: /leaders/**
      serviceId: gamification
      strip-prefix: false

eureka:
  client:
    service-url:
      default-zone: http://localhost:8761/eureka/

endpoints:
  routes:
    sensitive: false
```

## ZuulFallbackProvider configuration
```
@Configuration
public class HystrixFallbackConfiguration {

    @Bean
    public ZuulFallbackProvider zuulFallbackProvider() {
        return new ZuulFallbackProvider() {

            @Override
            public String getRoute() {
                // Might be confusing: it's the serviceId property and not the route
                return "multiplication";
            }

            @Override
            public ClientHttpResponse fallbackResponse() {
                return new ClientHttpResponse() {
                    @Override
                    public HttpStatus getStatusCode() throws IOException {
                        return HttpStatus.OK;
                    }

                    @Override
                    public int getRawStatusCode() throws IOException {
                        return HttpStatus.OK.value();
                    }

                    @Override
                    public String getStatusText() throws IOException {
                        return HttpStatus.OK.toString();
                    }

                    @Override
                    public void close() {}

                    @Override
                    public InputStream getBody() throws IOException {
                        return new ByteArrayInputStream("{\"factorA\":\"Sorry, Service is Down!\",\"factorB\":\"?\",\"id\":null}".getBytes());
                    }

                    @Override
                    public HttpHeaders getHeaders() {
                        HttpHeaders headers = new HttpHeaders();
                        headers.setContentType(MediaType.APPLICATION_JSON);
                        return headers;
                    }
                };
            }
        };
    }
}
```

Ref: https://netflixtechblog.com/announcing-evcache-distributed-in-memory-datastore-for-cloud-c26a698c27f7

# EVCache: Distributed in-memory datastore for Cloud

EVCache is a distributed in-memory caching solution based on memcached & spymemcached that is well integrated with Netflix OSS and AWS EC2 infrastructure.
Netflix has now opensourced it https://github.com/Netflix/EVCache.

EVCache is an abbreviation for:
* **Ephemeral** — The data stored is for a short duration as specified by its TTL (Time To Live).
* **Volatile** — The data can disappear any time (Evicted).
* **Cache** — An in-memory key-value store.

The advantages of distributed caching are:

* Faster response time compared to data being fetched from source/database
* Reduces the load and number of servers needed to handle the requests as most of the requests are served by the cache
* Increases the throughput of the services fronted by the cache

## What is an EVCache App?
* EVCache App is a logical grouping of one or more memcached instances (servers). Each instance can be a
* EVCache Server (to be open sourced soon) running memcached and a Java sidecar app
* EC2 instance running memcached
* ElastiCache instance
* instance that can talk memcahced protocol (eg. Couchbase, MemcacheDB)

Each app is associated with a name. Though it is not recommended, a memcached instance can be shared across multiple EVCache Apps.

## What is an EVCache Client?
EVCache client manages all the operations between an Java application and EVCache App.

## What is an EVCache Server?
EVCache Server is an EC2 instance running an instances of memcached and a Java Sidecar application. The sidecar is responsible for interacting with Eureka, monitoring the memcached process and collecting and reporting performance data to the Servo. This will be Open Sourced soon.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/ev_cache.png)

Figure 1 shows an EVCache App consisting of 3 memcached nodes with an EVCache client connecting to it.

The data is sharded across the memcached nodes based on Ketama consistent hashing algorithm. In this mode all the memcached nodes can be in the same availability zone or spread out across multiple availability zones.


## Multi-Cluster EVCache Deployment
Figure 2 shows an EVCache App in 2 Clusters (A & B) with 3 memcached nodes in each Cluster. Data is replicated between the two clusters. To achieve low latency, reliability and isolation all the EC2 instances for a cluster should be in the same availability zone. This way if an availability zone is having any issues the performance of the other zone is not impacted. In a scenario where we lose instances in one cluster, we can dynamically set that cluster to “write only” and direct all the read traffic to other zone. This ensures that latency and cache hit rate is not impacted.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/ev_cache2.png)


In the above scenario, the data is replicated across both the clusters and is sharded across the 3 instances in each cluster based on **Ketama** consistent hashing algorithm. All the reads (get, getBulk, getAndTouch) by a client are sent to the same zone whereas the writes(set & delete) are done on both the zones. The data replication across both the clusters increases its availability. Since the data is always read from the local zone this improves the latency. This approach is best suited if you want to achieve better performance with higher reliability.

If some data is lost due to an instance failure or eviction in a cluster, then the data can be fetched from the other cluster. Having fallback improves the both availability & reliability. In most cases fetching data from other cluster(fallback) is much faster than getting the data from source.

Figure 3 shows an EVCache App in 3 Clusters (A, B & C) with 3 EVCache servers in each Cluster. Each cluster is in an availability zones. An EVCache server (to be open sourced soon) consists of a memcached instance and sidecar app. The sidecar app interacts with Eureka Server and monitor the memcached process.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/ev_cache3.png)
Figure 3

In the above scenario the EVCache client gets the list of servers from Eureka and creates cluster based on the availability zone of each EVCache Server. If the EVCache Server instances are added or removed the EVCache client re-configures itself to reflect this change. This is transparent to the client.

Similar to Multi-Clustered deployment the data is sharded across the 3 instances within the same zone based on Ketama consistent hashing algorithm. All the reads by a client are performed on the same zone as the client whereas the writes are done across all the zones. This ensures that data is replicated across all the zones thus increasing its availability. Since the data is always read from the local zone this improves the latency at the same time improving the data reliability.

If zone fallback is enabled and some data is lost due to instance/zone failure or eviction, then the data can be fetched from the clusters in other zone. This however causes an increase in latency but higher reliability. In most cases fetching data from other zone is much faster than getting the data from source.



