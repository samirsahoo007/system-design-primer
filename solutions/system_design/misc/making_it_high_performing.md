What we cover in this post tries to explain in as simple terms as possible what it takes to optimise a web site so that it is lightning fast and scalable to handle very high numbers of visitors. The techniques we use are applicable to most applications our customers usually build their sites on, such as WordPress, Drupal, Joomla and Magento, amongst others.
Let's see how we can make our site's landing page to load quickly...

Redis => data caching
Varnish => frontend caching
CloudFlare => web static data(images, javascript, css) caching

**Redis and Varnish do server level caching whereas CloudFlare do network level caching**

# Redis:
For most use cases, we only need the tip - data caching. In simple terms, Redis can be used as a very fast way to cache data in memory. One of the pain points that make web sites slow is the time it takes to query a database, often where much of information necessary to display the page is stored. This is particularly true on busy sites that have lots of visitors, and make lots of queries to a database. Redis can be used to cache some or all of your database in memory, and querying data from memory takes a fraction of the time compared to querying from much slower hard disks.

So it's also really useful at speeding up web sites that may rely on external APIs as well. For instance on our own web site, we query various external APIs such as TrustPilot reviews, recent blog posts and Twitter. These are cached directly into Redis so that the data is available instantly, without needing to do a live query to the external API or even build a MySQL table to store that data in.


# Varnish:
Varnish is a web application accelerator, which sits in front of the web server. To understand what Varnish does, it's first important to understand what happens normally, without Varnish. When a visitor accesses your web site, the web server will receive a request for a page, which will then spawn more requests to find all the components needed to display your page. Typically, all your images, CSS files, Javascript, PHP scripts will be requested, PHP code will execute, MySQL queries will be made and data returned, and eventually, the components and resulting HTML will be downloaded to your browser. That's an awful lot of work. When you've got lots of people accessing your web site, your web server is working overtime trying to deliver all of these requests, to all of these visitors. Often, it's doing all of this work to generate the exact same content for different visitors.

With Varnish, all that hard work only needs to be done once, or at least only until the content or page itself changes. Once it's got all the necessary data to display the web page, Varnish caches that data in memory. Now, when the next visitor arrives, your web server can put its feet up and let Varnish simply return the ready made page and all the components in milliseconds, without a single request to disk, MySQL query or PHP execution. This means that your web server only needs to do the hard work when the requested data has changed, freeing it up to handle many times more requests than it would otherwise have been capable.

Varnish is fast, really fast. It typically speeds up delivery by a factor or 300 - 1000 x, and will allow your server to handle huge volumes of traffic. No need to buy a bigger boat, we're now flying a plane.


# CloudFlare:
So both Varnish and Redis are server level caching solutions, software that sits on your server, making things super snappy. By contrast, CloudFlare is a network level caching layer, and a big one at that. Let's say your VPS is in our UK datacentre, but you've got lots of visitors to your web site in the US. Normally when they load your web site, all of the components that make it up will travel across fiberoptic cables under the Atlantic. Now this happens incredibly quickly, but it would be much quicker if that data didn't have to travel all those thousands of miles. Enabling CloudFlare means that your data can be cached in a datacentre that's closer to each of your visitors. The CloudFlare network is vast, with over 100 datacentres spanning every continent on Earth.

By default, CloudFlare will only cache your static data - for instance your images, javascript and CSS - often what forms the bulk of your web site size. It can, with configuration, also cache your web site page content.

There are two major benefits to this network level caching.

* Firstly, your web site will load incredibly quickly for your visitors wherever they are in the world. Your site should load pretty much as quickly for someone in Australia, Brazil or Japan as it does for someone in the UK (assuming that's where your Kualo server is).

* Secondly, every request that CloudFlare serves, is one less request that your server has to handle. The fewer requests your server handles, the less powerful that server has to be to serve the same amount of visitors. So this means even if you don't have an international audience, CloudFlare will nonetheless take extra strain off your server and free up more resources so you can handle more visitors for the same or less money.

In the example above, the orange portion represents all the requests for a web site that were served from CloudFlare's network, and the blue portion represents requests that were passed to the server. Over 86% of the requests for this site over that 24 hour period came from CloudFlare's network, meaning that your server can be much smaller and cheaper and still allow you to handle all those visitors. If you consider that many of those requests to the server may well have been powered by Varnish, maybe only 1-10% of those requests to the server would have needed to have been processed by the web server itself. That means only a tiny amount of requests needed to run PHP or make a MySQL query - and the ones that do are going to be even faster than normal because of Redis.

The picture gets even better when you consider the bandwidth savings. Any data thats served from CloudFlare saves you bandwidth - you can expect your bandwidth reduction to go down by a similar percentage as your requests, if not even more.

# Railgun
Railgun makes web sites even faster. 
When CloudFlare requests data from your server, because that data has either changed or didn't already exist in that locations cache, it is transmitted to CloudFlare and onto the end user much faster than it would normally.

What's more, Railgun speeds up re-caching of dynamic content by only transmitting what's changed. Let's say you have an e-commerce web site and you change out a featured product on the home page. Railgun will compare the cached version that it has already stored with the updated version, and will only send the bytes or even bits of data needed to effect that change on the resulting web page. This results in an average 200% additional performance increase.

Again, we're only touching the tip of the iceberg, and there are a myriad of other features and advantages in CloudFlare that can add speed and security benefits. With CloudFlare and Railgun in the mix, we're no longer just flying any old plane - we're on a rocket.


Ref: https://www.kualo.in/blog/fast-scalable-web-sites-with-redis-varnish-cloudflare-railgun

