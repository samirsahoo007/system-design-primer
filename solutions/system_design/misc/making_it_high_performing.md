
Redis => data caching
Varnish => frontend caching
CloudFlare => web static data(images, javascript, css) caching


What we cover in this post tries to explain in as simple terms as possible what it takes to optimise a web site so that it is lightning fast and scalable to handle very high numbers of visitors. The techniques we use are applicable to most applications our customers usually build their sites on, such as WordPress, Drupal, Joomla and Magento, amongst others.
The first impression of a good WordPress site is that its site’s landing page should load quickly. Then, the loading times of rest of interlinked pages should be fast. For a website to load fast it should have different layered approaches to caching.

# How to improve page load times of website?
* The first layer is the presence of content delivery network (CDN).
* The second layer of creating the faster website is to install a free TLS/SSL certificate signed by CloudFlare on your origin server.
* * It is completely free of cost.
* Enable Varnish cache and enable Varnish cache settings for individual domains.
* Most of times sites in the web browser wats for around 1.6 seconds for favicon to load. It is a good idea to add the favicon to reduce this waiting time of web browser in visitor’s computer.
* Enable HTTP/2on CloudFlare to work with Varnish cache.
* Use HTTPS protocol on your website.

**Redis and Varnish do server level caching whereas CloudFlare do network level caching**

## Redis:
For most use cases, we only need the tip - data caching. In simple terms, Redis can be used as a very fast way to cache data in memory. One of the pain points that make web sites slow is the time it takes to query a database, often where much of information necessary to display the page is stored. This is particularly true on busy sites that have lots of visitors, and make lots of queries to a database. Redis can be used to cache some or all of your database in memory, and querying data from memory takes a fraction of the time compared to querying from much slower hard disks.

So it's also really useful at speeding up web sites that may rely on external APIs as well. For instance on our own web site, we query various external APIs such as TrustPilot reviews, recent blog posts and Twitter. These are cached directly into Redis so that the data is available instantly, without needing to do a live query to the external API or even build a MySQL table to store that data in.


## Varnish:
Varnish is a web application accelerator, which sits in front of the web server. To understand what Varnish does, it's first important to understand what happens normally, without Varnish. When a visitor accesses your web site, the web server will receive a request for a page, which will then spawn more requests to find all the components needed to display your page. Typically, all your images, CSS files, Javascript, PHP scripts will be requested, PHP code will execute, MySQL queries will be made and data returned, and eventually, the components and resulting HTML will be downloaded to your browser. That's an awful lot of work. When you've got lots of people accessing your web site, your web server is working overtime trying to deliver all of these requests, to all of these visitors. Often, it's doing all of this work to generate the exact same content for different visitors.

With Varnish, all that hard work only needs to be done once, or at least only until the content or page itself changes. Once it's got all the necessary data to display the web page, Varnish caches that data in memory. Now, when the next visitor arrives, your web server can put its feet up and let Varnish simply return the ready made page and all the components in milliseconds, without a single request to disk, MySQL query or PHP execution. This means that your web server only needs to do the hard work when the requested data has changed, freeing it up to handle many times more requests than it would otherwise have been capable.

Varnish is fast, really fast. It typically speeds up delivery by a factor or 300 - 1000 x, and will allow your server to handle huge volumes of traffic. No need to buy a bigger boat, we're now flying a plane.


## CloudFlare:
So both Varnish and Redis are server level caching solutions, software that sits on your server, making things super snappy. By contrast, CloudFlare is a network level caching layer, and a big one at that. Let's say your VPS is in our UK datacentre, but you've got lots of visitors to your web site in the US. Normally when they load your web site, all of the components that make it up will travel across fiberoptic cables under the Atlantic. Now this happens incredibly quickly, but it would be much quicker if that data didn't have to travel all those thousands of miles. Enabling CloudFlare means that your data can be cached in a datacentre that's closer to each of your visitors. The CloudFlare network is vast, with over 100 datacentres spanning every continent on Earth.

In addition to this **CloudFlare works behind the scenes to block known security threats.  Abusive bots and crawlers will be automatically limited from wasting your bandwidth and server resources**.
When your site is protected by CloudFlare your web pages will be cached and optimized for speedy delivery so your visitors get some of the fastest page load times and best performance they have ever seen!
![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/maxresdefault.jpg)

By default, CloudFlare will only cache your static data - for instance your images, javascript and CSS - often what forms the bulk of your web site size. It can, with configuration, also cache your web site page content.

There are two major benefits to this network level caching.

* Firstly, your web site will load incredibly quickly for your visitors wherever they are in the world. Your site should load pretty much as quickly for someone in Australia, Brazil or Japan as it does for someone in the UK (assuming that's where your Kualo server is).

* Secondly, every request that CloudFlare serves, is one less request that your server has to handle. The fewer requests your server handles, the less powerful that server has to be to serve the same amount of visitors. So this means even if you don't have an international audience, CloudFlare will nonetheless take extra strain off your server and free up more resources so you can handle more visitors for the same or less money.

In the example above, the orange portion represents all the requests for a web site that were served from CloudFlare's network, and the blue portion represents requests that were passed to the server. Over 86% of the requests for this site over that 24 hour period came from CloudFlare's network, meaning that your server can be much smaller and cheaper and still allow you to handle all those visitors. If you consider that many of those requests to the server may well have been powered by Varnish, maybe only 1-10% of those requests to the server would have needed to have been processed by the web server itself. That means only a tiny amount of requests needed to run PHP or make a MySQL query - and the ones that do are going to be even faster than normal because of Redis.

The picture gets even better when you consider the bandwidth savings. Any data thats served from CloudFlare saves you bandwidth - you can expect your bandwidth reduction to go down by a similar percentage as your requests, if not even more.

## Railgun
Railgun makes web sites even faster. 
When CloudFlare requests data from your server, because that data has either changed or didn't already exist in that locations cache, it is transmitted to CloudFlare and onto the end user much faster than it would normally.

What's more, Railgun speeds up re-caching of dynamic content by only transmitting what's changed. Let's say you have an e-commerce web site and you change out a featured product on the home page. Railgun will compare the cached version that it has already stored with the updated version, and will only send the bytes or even bits of data needed to effect that change on the resulting web page. This results in an average 200% additional performance increase.

Again, we're only touching the tip of the iceberg, and there are a myriad of other features and advantages in CloudFlare that can add speed and security benefits. With CloudFlare and Railgun in the mix, we're no longer just flying any old plane - we're on a rocket.

### Understanding Railgun:
When you host your site with DreamHost, we pick a server for you in one of our datacenters. Currently we have two datacenters in California and one in Virginia. This probably works just fine if you happen to be in the United States. However, your website is still limited to just one server. CloudFlare on the other hand has 23 data centers spread out across the world and is adding more all the time. These datacenters act like a CDN (Content Delivery Network) for sites using the service. In other words, CloudFlare will automatically make copies of your website across the world, and when someone visits your site they’ll automatically go to the closest one.

When CloudFlare is enabled, your site visitors are automatically routed through CloudFlare. CloudFlare acts as a CDN, and provides other features such as automatically optimizing pages.

That sounds great, but you may still be wondering how all that actually works. I don’t know all the intricate details of what CloudFlare does with their servers, but I can tell you what we do here at DreamHost. Under the hood there are three things we use:

#### DNS — 
We modify the DNS records for your site to point to CloudFlare instead of the DreamHost servers. DNS stands for Domain Name System. In layman’s terms, DNS is how computers convert a domain name such as dreamhost.com to an IP address like 208.97.187.204 (every computer on the internet has an IP address). Most people never need to worry about DNS records, and as long as you’re using DreamHost for your DNS servers, we handle all the necessary changes for you when setting up CloudFlare.

#### CloudFlare’s Hosting Provider API — 
We make use of CloudFlare’s Hosting Provider API to set up your account with them. If you’re a software developer, you’re probably familiar with the term API (Application Programming Interface). I won’t really go into the details about how the CloudFlare API works; this is just a system we use to communicate and synchronize settings with CloudFlare.

#### mod_cloudflare — 
mod_cloudflare is an Apache module that allows your server to know the true IP address of your site visitors even when the connection (from DreamHost’s perspective) is coming from CloudFlare. Apache is the software we use for our web servers, and an “Apache module” is just a plugin for that software. There’s nothing all that magical about how this works. CloudFlare’s servers send a little bit of extra data using HTTP headers, and mod_cloudflare reads the headers and fixes up the reported IP address in Apache.
And… that’s it. Everything else with regards to your website will be the same as it always has been. You still have the option of being on our Shared, VPS, or Dedicated Hosting. You don’t need to install any special plugins or make any changes to your site (the only special plugin is mod_cloudflare, and we’ve already installed that on all of our servers — even if you’re not using CloudFlare).

Oh, I almost forgot… RAILGUN!!! Yes, that was supposed to be the subject of this post!

People using CloudFlare up until now may have noticed that the connection between DreamHost and CloudFlare (aka the internet) is still a potential bottleneck. This can especially be a problem for sites with dynamic content because that dynamic content would need to be retrieved from the DreamHost web server each time. This is where Railgun makes a difference.

Railgun compresses the data in the connection between DreamHost and CloudFlare in order to minimize the amount of data we need to transmit over the internet. Railgun does this by taking advantage of the fact that even pages with dynamic content often only have small parts that change. Rather than transmitting a new copy of the entire page over the internet every time, Railgun transmits only the parts that changed.

Instead of connecting directly to CloudFlare, the web server’s connection is routed through a Railgun Server, which compresses the data before it goes over the internet. The load on the webserver remains the same, but the amount of data transmitted over the internet is greatly reduced.
Less data going over the internet (between DreamHost and CloudFlare’s servers) means your site loads faster, and we can save bandwidth for things that are important...

If you're currently a DreamHost customer, check out this wiki page for instructions on how to enable Railgun on your site. Even if you don’t use DreamHost, we still recommend taking advantage of the service CloudFlare provides. CloudFlare should work with any web host, although not all of them may support Railgun.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/cloudflare_no_railgun.png)

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/cloudflare_railgun.png)


Ref: https://www.dreamhost.com/blog/cloudflare-railgun/
Ref: https://www.kualo.in/blog/fast-scalable-web-sites-with-redis-varnish-cloudflare-railgun

### Can I use Cloudflare and Varnish together?
You may also modify vcl\_recv to strip the \_\_cfduid cookies set by Cloudflare so Varnish can cache the response. The following VCL will strip all cookies starting with two underscores or including 'has\_js', such as Cloudflare and Google Analytics cookies:

```
    sub vcl_recv {
      # Remove has_js and Cloudflare/Google Analytics __* cookies.
      set req.http.Cookie = regsuball(req.http.Cookie, "(^|;\s *)(_[_a-z]+|has_js)=[^;]*", "");
      # Remove a ";" prefix, if present.
      set req.http.Cookie = regsub(req.http.Cookie, "^;\s*", "");
    }
```

#### About Varnish
Varnish Software’s powerful caching technology helps the world’s biggest content providers deliver lightning-fast web and streaming experiences for huge audiences, without downtime or loss of performance.

Our solutions combine open-source flexibility with enterprise robustness to speed up media streaming services, accelerate websites and APIs, and enable global businesses to build custom CDNs, unlocking unbeatable content delivery performance and resilience.

#### About CloudFlare
Cloudflare is a successful combination of network security and performance solution that increases websites or networks speed and provides protection of websites, mobile applications, APIs, and SaaS services. It is one of the leading Content Delivery Network on the market.

This CDN service speeds up and enhances the performance of networks, websites, apps, and APIs. It operates as a network of proxy servers and data centers (more than 100) that are located around the world, powering over 10 trillion requests per month. The service can manage 10Tbps in bandwidth for their users.

When it comes to security, the key security elements include protection against SQL injection, which attacks the code of the website. It also protects from Distributed Denial of Service (DDoS) attacks, which is on the rise by exploitation of insecure Internet-of-Things devices. These downtimes can make the average hourly cost as high as $100,000 per hour.

# How to enable Varnish cache on your origin server?
* Log in to cPanel
* Provide User Name and Password
* Go to WEB ACCELERATOR in paper_lantern theme.
* Click on MANAGE VARNISH
* From MANAGE VARNISH SETTINGS scroll down to CACHE SETTINGS FOR INDIVIDUAL DOMAINS.
* Enable it on your DOMAIN
* It might take 30 minutes to enable Varnish cache on your website.

# What is Varnish Cache?:
* Varnish Cache speeds up website rendering time to three to five times.
* It caches static and dynamic contents of WordPress website.
* Dynamic contents of a website include images, CSS (Cascade style sheets), plain HTML.
* Varnish Cache is a web application accelerator.
* It works as caching HTTP reverse proxy.
* It is installed on your origin server. (e.g. cPanel)
* If your network speed of hosting server is fast then it works really fast.
* It can deliver the high level of content through regular off-the-shelf hardware.

# How to enable HTTP/2 on your website?
* Set up Free CloudFlare CDN for your website
* Install a free TLS certificate (SSL) signed by CloudFlare on your origin server.
* Login to CloudFlare and then click on your website name in its dashboard.
* Go to NETWORK and then HTTP/2 and enable it. Most of the times it is enabled by default if not enable it there.

# What is HTTP/2?
* It is the prominent revision of World Wide Web (WWW)
* It improves the way HTTP requests and responses are sent through low-latency transport of web contents.
* It increases page load times of website.
* It creates per-domain multiplexing of retrieving resources.
* HTTP header compression.
* It provides server push by removing latency times on web browser.
* It works nicely when Varnish cache is enabled on the origin server.

# How to add the favicon to the website?
* Log in to WordPress Admin
* Put Username or Email Address and password.
* From sidebar of WordPress Admin choose APPEARANCE
* Then CUSTOMISE. I USE Twenty Twelve theme.
* Then go to SITE IDENTITY
* Then go to SITE ICON. It is favicon.
* Here upload your favicon and save website.

After enabling CloudFlare, SSL and HTTP/2 on my site, I saw the mobile speed of the website is 77 and desktop is 81. After implementation of Varnish cache, now the mobile score of website 79 and desktop score of the website is 91. This is from Google PageSpeed insight.
I introduced CloudFlare CDN as one cache layer for the website and then Varnish Cache as another layer of caching of contents. This makes the website faster. In this way, original server or the host of the website is being kept secure and the handling of large data traffic rests with reverse proxy Varnish and content distribution network CloudFlare.

Ref: https://mohanmekap.medium.com/how-to-use-varnish-and-cloudflare-for-maximum-caching-d3a7c42e54c3

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/MiddleHost-mhCache-Varnish-Cache-for-wordpress.png)

mhCache => Middlehost cache(e.g. Varnish)

# General best practices for load balancing at your origin with Cloudflare

When integrating with Cloudflare, please consider the following for best practices when utilizing load balancers within your website's host environment:

* Cloudflare's DNS Load Balancing 
* HTTP keep-alive
* Session cookies
* Railgun 

## DNS Load Balancing

Cloudflare's Load Balancing feature supports DNS-based load balancing with active health checks against your origin servers. It expands on Cloudflare's existing Anycast DNS network to provide DDoS-resilient failover (steering around unhealthy origins) and geo-steering (directing users to specific pools of origins).

## HTTP keep-alive (HTTP persistent connection)

Cloudflare maintains keep-alive connections to improve performance and reduce cost of recurring TCP connects in the request transaction as Cloudflare proxies customer traffic from its edge network to the site's origin.

Ensure HTTP Keep-Alive connections are enabled on your origin. Cloudflare reuses open TCP connections for up to 15 minutes (900 seconds) after the last HTTP request. Origins close TCP connections if too many are open. HTTP Keep-Alive helps avoid premature reset of connections for requests proxied by Cloudflare.

## Session cookies

If using HTTP cookies to track and bind user sessions to a specific application server at the load balancer, it is best is to configure the load balancer to parse HTTP requests by cookie headers and directing each request to the correct application server even if HTTP requests share the same TCP connection due to keep-alive.

For example: F5 BIG-IP load balancers will set a session cookie (if none exists) at the beginning of a TCP connection and then ignore all cookies passed on subsequent HTTP requests made on the same TCP socket. This tends to break session affinity because Cloudflare will send multiple different HTTP sessions on the same TCP connection. (HTTP cookie-based session affinity).

## Railgun (WAN Optimization)

The ideal setup when using Railgun and a load balancer is to place the Railgun listener in front of the load balancer. Placing the listener in front is the ideal setup, as that allows the LB to handle the HTTP/S connections as normal, as it is difficult to load balance the long-lived TLS connection between the sender/listener.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/hc-import-railgun_diagram_lb_setup.png)

Ref: https://support.cloudflare.com/hc/en-us/articles/212794707-General-best-practices-for-load-balancing-at-your-origin-with-Cloudflare

# What is a CDN edge server?

A CDN edge server is a computer that exists at the logical extreme or “edge” of a network. An edge server often serves as the connection between separate networks. A primary purpose of a CDN edge server is to store content as close as possible to a requesting client machine, thereby reducing latency and improving page load times.

An edge server is a type of edge device that provides an entry point into a network. Other edges devices include routers and routing switches. Edge devices are often placed inside Internet exchange points (IxPs) to allow different networks to connect and share transit.

## How does an edge server work?

In any particular network layout, a number of different devices will connect to each other using one or more predefined network pattern. If a network wants to connect to another network or the larger Internet, it must have some form of bridge in order for traffic to flow from one location to another. Hardware devices that creates this bridge on the edge of a network are called edge devices.

**Networks connect across the edge**

In a typical home or office network with many devices connected, devices such as mobile phones or computers connect and disconnect to the network through a hub-and-spoke network model. All of the devices exist within the same local area network (LAN), and each device connects to a central router, through which they are able to connect with each other.

In order to connect a second network to the first network, at some point the connection must be made between the networks. The device through which the networks are able to connect with each other is, by definition, an edge device.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/cdn-edge-network-device.png)
CDN edge network device diagram

Now, if a computer inside Network A needs to connect to a computer inside Network B, the connection must pass from network A, across the network edge, and into the second network. This same paradigm also works in more complex contexts, such when a connection is made across the Internet. The ability for networks to share transit is bottlenecked by the availability of edge devices between them.

When a connection must traverse the Internet, even more intermediary steps must be taken between network A and network B. For the sake of simplicity, let's imagine that each network is a circle, and the place in which the circles touch is the edge of the network. In order for connection to move across the Internet, it will typically touch many networks and move across many network edge nodes. Generally speaking, the farther the connection must travel, the greater the number of networks that must be traversed. A connection may traverse different Internet service providers and Internet backbone infrastructure hardware before reaching its target.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/cdn-edge-server-placement.png)
CDN edge server placement diagram

A CDN provider will place servers in many locations, but some of the most important are the connection points at the edge between different networks. These edge servers will connect with multiple different networks and allow for traffic to pass quickly and efficiently between networks. Without a CDN, transit may take a slower and/or more convoluted route between source and destination. In worst case scenarios, traffic will “trombone” large distances; when connecting to another device across the street, a connection may move across the country and back again. By placing edge servers in key locations, a CDN is able to quickly deliver content to users inside different networks. To learn more about the improvements of using CDN, explore how CDN performance works.

## What is the difference between an edge server and an origin server?

An origin server is the web server that receives all Internet traffic when a web property is not using a CDN. Using an origin server without a CDN means that each Internet request must return to the physical location of that origin server, regardless of where in the world it resides. This creates an increase in load times which increases the further the server is from the requesting client machine.

CDN edge servers store (cache) content in strategic locations in order to take the load off of one or more origin servers. By moving static assets like images, HTML and JavaScript files (and potentially other content) as close as possible to the requesting client machine, an edge server cache is able to reduce the amount of time it takes for a web resource to load. Origin servers still have an important function to play when using a CDN, as important server-side code such as a database of hashed client credentials used for authentication, typically is maintained at the origin. Learn about the Cloudflare CDN with edge servers all over the globe.

# What is a data center?

A data center is a facility housing many networked computers that work together to process, store, and share data. Most major tech companies rely heavily upon data centers as a central component in delivering online services.

## What is the difference between a data center and a point-of-presence (PoP)?

The terms data center and point-of-presence (PoP) are sometimes used interchangeably, though distinctions can be made between them. Speaking generally, a PoP may refer to a company having a single server presence in a location while a data center may refer to a location that houses multiple servers. Instead of referring to multiple PoPs in one location, Cloudflare uses the term data center to indicate a location in which many of our servers are maintained.

The concept of a point-of-presence rose to prominence during the court ordered breakup of the Bell telephone system. In the court decision, a point-of-presence referred to a location where long-distance carriers terminate services and shift connections onto a local network. Similarly, on the modern Internet a PoP typically refers to where CDNs have a physical presence in a location, often in the junctures between networks known as Internet exchange points (IxP).

A data center refers to a physical location in which computers are networked together in order to improve usability and reduce costs related to storage, bandwidth, and other networking components. Data centers such as IxP co-location facilities allow different Internet service providers, CDN’s, and other infrastructure companies to connect with each other to share transit.

