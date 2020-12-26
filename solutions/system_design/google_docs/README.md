**Collaborative editing** is a process of writing and editing documents or projects by more than one person. Collaborative editing software tools like Firepad and Textflow as well as services like Google Docs facilitate collaboratively editing individual computer files by multiple users.

e.g. Google Docs, Etherpad, Mocking Bird.

We can think that we can apply lock on a particular user until he completes editing and then control can be shifted to other users. But it may result in high Bandwidth usage and also cause some latency issues. So we need to have a **lock-free architecture**.

# Approach 1:- Operational Transformation:

Operational transformation/Event Passing, at its core, is an optimistic concurrency control mechanism.it allows two editors to modify the same section of a document at the same time without conflict. or rather, it provides a mechanism for sanely resolving those conflicts so that neither user intervention nor locking becomes necessary.

For every Insert(),Delete(),Update(),Retain() operations both users will make api calls and server gets triggered

In terms of conflicts it is:

1. stable
2. Reliable

# Approach 2:- Differential Synchronisation

It works the same as like git diff:

git diff is a multi-use Git command that when executed runs a diff function on Git data sources. These data sources can be commits, branches, files, and more. ... The git diff command is often used along with git status and git log to analyze the current state of a Git repo.

In the same way, it takes a bunch of data after every diff and sends it to other users.

**Strategies:-**

Let us assume the doc had characters "AT" in it now clients perform following operations

**Client 1 Client 2**

Delete("T",1) Insert("H",0)

o/p:- A HAT

Then there comes a conflict. To encounter it OT(Operational Transformation) follows the following function

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/google_docs1.png)

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/google_docs2.png)

a- delete("T",1)--->A

b-insert("H",0)---->HAT

Now a' and b' will replicate the properties of a and b in the following way.

a'-delete("T",2) --->HA

b'-insert("H",0) --->HA

In the above case, OT will adjust a',b' in such a way that it doesn't encounter any conflict.

Finally, both the users will be having same data in their shared doc.

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/designs/google_docs_simplified.jpg)

* Here the idea is to use the microservice-based architecture where in the starting stage the clients will be contacting the API gateway for any requests and then we can use WebSockets.
* We can use Kubernetes in place of gateway since it provides cool features like security, single entry point & API composition, security, Dynamic service discovery, circuit breaking.
* NoSQL + Redis will gonna be a crazy combination which gonna be Realtime and Lightweight.
* We can also add Email/GCM Notifications service at API gateway so that it can be a more flexible and cool feature to users.
* The Operational queue keeps track of current ongoing operation since not only type the characters it also contains features like font change, using templates, etc.,
* Sessions server keeps track of time in and time out actions by using time-series DB.

