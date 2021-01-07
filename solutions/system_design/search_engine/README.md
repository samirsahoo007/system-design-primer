Also read it
https://www.google.com/search/howsearchworks/


**Search Engine** refers to a huge database of internet resources such as web pages, newsgroups, programs, images etc. It helps to locate information on World Wide Web.
User can search for any information by passing query in form of keywords or phrase. It then searches for relevant information in its database and return to the user.

**Google-scale search engine**
Google has an estimated index size of 50 billion web pages , selected from the 130 Trillion web pages the World Wide Web consists of (How Search Works).
Google serves the answer to a search query within 0.2 seconds on average.
Google answers 300,000 queries in parallel ( Google now handles 2 trillion searches per year ) during that 0.2 seconds.


## Search Engine Components
Generally there are three basic components of a search engine as listed below:

* Web Crawler
* Database
* Search Interfaces

### Web crawler
It is also known as **spider or bots**. It is a software component that traverses the web to gather information.

### Database
All the information on the web is stored in database. It consists of huge web resources.

### Search Interfaces
This component is an interface between user and the database. It helps the user to search through the database.

## Search Engine Working
Web crawler, database and the search interface are the major component of a search engine that actually makes search engine to work. Search engines make use of Boolean expression AND, OR, NOT to restrict and widen the results of a search.

The first phase of implementing Google (or any search engine) is to build an indexer.
To implement this, consider two parts: a crawler and indexer.

The web crawler's job is to spider web page links and dump them into a set. The most important step here is to avoid getting caught in infinite loop or on infinitely generated content. Place each of these links in one massive text file (for now).

Second, the indexer will run as part of a Map/Reduce job. (Map a function to every item in the input, and then Reduce the results into a single 'thing'.) The indexer will take a single web link, retrieve the website, and convert it into an index file. (Discussed next.) The reduction step will simply be aggregating all of these index files into a single unit. (Rather than millions of loose files.) Since the indexing steps can be done in parallel, you can farm this Map/Reduce job across an arbitrarily-large data center.

The next part is explaining how you can compute meaningful results. The short answer here is 'a lot more Map/Reduces', but consider the sorts of things you can do:

* For each web site, count the number of incoming links. (More heavily linked-to pages should be 'better'.)
* For each web site, look at how the link was presented. (Links in an < h1 > or < b > should be more important than those buried in an < h3 >.)
* For each web site, look at the number of outbound links. (Nobody likes spammers.)
* For each web site, look at the types of words used. For example, 'hash' and 'table' probably means the web site is related to Computer Science. 'hash' and 'brownies' on the other hand would imply the site was about something far different.

The final phase is actually serving the results. Hopefully you've shared some interesting insights in how to analyze web page data, but the question is how do you actually query it? Anecdotally 10% of Google search queries each day have never been seen before. This means you cannot cache previous results.

You cannot have a single 'lookup' from your web indexes, so which would you try? How would you look across different indexes? (Perhaps combining results -- perhaps keyword 'stackoverflow' came up highly in multiple indexes.)

Also, how would you look it up anyways? What sorts of approaches can you use for reading data from massive amounts of information quickly? (Feel free to namedrop your favorite NoSQL database here and/or look into what Google's BigTable is all about.) Even if you have an awesome index that is highly accurate, you need a way to find data in it quickly. (E.g., find the rank number for 'stackoverflow.com' inside of a 200GB file.)


Following are the steps that are performed by the search engine:

* The search engine looks for the keyword in the index for predefined database instead of going directly to the web to search for the keyword.

* It then uses software to search for the information in the database. This software component is known as web crawler.

* Once web crawler finds the pages, the search engine then shows the relevant web pages as a result. These retrieved web pages generally include title of page, size of text portion, first several sentences etc.

These search criteria may vary from one search engine to the other. The retrieved information is ranked according to various factors such as frequency of keywords, relevancy of information, links etc.
* User can click on any of the search results to open it.

## Architecture
Most large scale search engines are based on an **Inverted index**.
For every term which occurs in the corpus (the web pages to be indexed) there is a separate posting list maintained.
Such a posting list contains pointers (Document ID / DocID) to all web pages where the specific term occurs within the text.

Together with the DocID there are usually also a rank, the number and the positions of the occurrences stored in the posting list.

The search engine architecture comprises of the three basic layers listed below:

* Crawling(Content collection and refinement).
* Indexing(Storing every occurrence of a term to its posting list within the inverted index.)
* Search

During search those web pages are identified which contain all query terms (for Boolean AND).
First the posting lists for all query terms are loaded from the inverted index.
Then the posting lists are intersected to find those DocID which occur in all posting lists .
Then the matching DocID have to be sorted by the rank.

For the top-10 ranked DocID the documents (web pages) have to be loaded from the document store.
From those 10 web pages the final results are generated and formatted with title, summary, Url and transferred back to the user.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/internet-search_engine_architecture.jpg)

## Search Engine Processing
**Indexing Process**
Indexing process comprises of the following three tasks:

* Text acquisition
* Text transformation
* Index creation

*Text acquisition*
It identifies and stores documents for indexing.

*Text Transformation*
It transforms document into index terms or features.

*Index Creation*
It takes index terms created by text transformations and create data structures to suport fast searching.

**Query Process**
Query process comprises of the following three tasks:

* User interaction
* Ranking
* Evaluation

*User interaction*
It supporst creation and refinement of user query and displays the results.

*Ranking*
It uses query and indexes to create ranked list of documents.

*Evaluation*
It monitors and measures the effectiveness and efficiency. It is done offline.

Ref: https://www.tutorialspoint.com/internet_technologies/search_engines.htm


## Scaling Issues

Length of the Posting List:

The maximum length of a posting list (number of docids) is almost equal to the number of indexed documents.

Explanation: The frequency of terms in a document follows Zip’s law Zipf's law

"The" as the most frequent word accounts for nearly 7% of all word occurrences (in English). It occurs several times per document (on average).

For an estimated Index size of 50 billion web pages and a query of "The Who" we would need two Posting Lists of 50 billion DocID each to load from disk and intersect them.

Assuming 8 byte for each DocID + rank + number & position of word occurrences, that would be 2 * 50 * 8 = 800 GB of data to load from disk and 2 * 50=100 billion DocID to intersect within 0.2 seconds.

## Concurrent queries

And then multiply the processing of 100 billion DocID for a single query with 300,000 queries in parallel during 0.2 seconds.

300,000 queries * 100 billion DocID have to be loaded from disk, kept in RAM and intersected on the processor in parallel within 0,2 seconds!

This naïve approach obviously does not scale!

## Scaling solutions:

* Distribute the load of a single query over multiple servers (e.g. 1000) with a doc partitioned index, where every server indexes only a fraction of the web pages. The partial indexes of the 1000 servers are queried in parallel and the results are aggregated to find the top 10 which are returned to the user.
* That’s only for a single query, to serve 300,000 queries in parallel there are an estimated 1 million servers used in multiple data centers.
* Keep the index in RAM and/or SSD instead of HDD.
* Compress the posting lists to reduce the size and load time. What are the most efficient ways to compress an inverted index?
* An additional bi-gram index, with much shorter posting lists which makes intersection between the two terms unnecessary. Exploits the fact that very frequent terms (Stop words) have only discriminatory power for phrase queries.

### Further scaling issues

#### Crawling (50 billion web pages within 1 year)

* download/io bandwidth for fetching
* The URL frontier : number of detected urls grows faster than number of crawled/index web pages (as every page contains several outgoing links) until half of the web is indexed.
* processor load for parsing/html stripping
* disk/io bandwidth for storing raw content (for later snippet generation)
* disk/io bandwidth for storing/checking detected urls for optimum crawling policy/strategy: to ensure a complete, polite, deadlocks/trap/circle free, non-duplicate crawling

#### Indexing (50 billion web pages within 1 year)

* disk/io write bandwidth
* index space
* slow random write vs. fast sequential write ( Log-structured merge-tree )
* processor load for sorting/compressing/ranking many huge posting lists

#### Searching (2 trillion queries per year; 300,000 in parallel)

* number of parallel searches
* random access to disk (seek time)
* reading of huge posting list from disk (read time/IO limit)
* processor load for intersection/ranking of huge posting lists
* denial of service protection / rate limiter / throttling


Google has never had millions of searches at the same time, their queries -per-second are currently about 63000 queries/sec in 2019. That number can be a few times more at peak loads, but not millions.

Google has been using **document-partitioned** system since about 2000 meaning a single machine is responsible for a portion of the entire Web and it returns best answers it has to another machine in charge of assembling answers at run time. That machine picks the best answers, chooses ads and other stuff and assembles all that to be returned as the result.

The Web and its index are really not that big - for 100B pages, which include both the fast and the slow parts of the index it takes a few KB per page for a total of say 300–500TB. In 2019 that is not that special and has not been in years. Currently 64GB memory sticks are available so a typical 1U machine with, say 24 memory slots, can be stuffed with 1536GB of RAM. A few hundred of those would suffice per cluster, which is a single replica of the entire Web.

It used to be the case that a single cluster/replica could serve about 100 queries/sec but that was way back with hard drives, everything in RAM changes that drastically and improves it to (many) thousand qps.

Note also that the fast portion of the index is only a fraction, perhaps 10B or so. But I do not believe Google cares much about saving money on all that.

Basically Google, as well as Facebook, Twitter and everybody else keeps everything in RAM. It is easy and cheap for them, and actually for others too as you can pick a 1TB of good older RAM (e.g. PC3–14900R of PC3–12800R) for as little as $600. You could stuff a solid fast index in less than 100TB for say < $100K together with the machines to put the memory.

But the main story now is that keyword based is a legacy technology, more than 20 years old. It is quaint to think that relevant results should just give some blue links to pages where keywords appear. The result should be a list of authoritative direct answers to your question.

Indeed Google tries to do that when it shows you a so-called info-box at the top of results but that works only for a small portion of the results and the actual results leave a lot to be desired.

The latest BERT update is about that, but is only a small step as acknowledged by Google themselves. It also only a partial step as BERT works only on short snippets supplied by traditional keyword search. Obviously we would to do everything end-to-end by nearest-neighbor-search in vector spaces for both questions and answers.

The requirements for such a system supporting tens of thousands qps would be a very different animal and Google has barely started there. The race is on who will get there first.

Another search engine example: https://www.gigablast.com/

**What I came to know that**
* Most query results are cached as static pages.
* Google has more than 1 million servers.

