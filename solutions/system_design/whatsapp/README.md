![alt text](https://miro.medium.com/max/1174/1*Ny0Ved3qrBZUTYMTUkUb9Q.png)

When two clients(A and B) want to communicate or send messages to each other, they first know the address of each other(It may be IP,MAC or any customized unique identity) and they exchange messages with each other over a network, in this case it is INTERNET.

# 1. Simple communication can be seen as below

![alt text](https://github.com/samirsahoo007/system-design-primer/blob/master/images/whatsapp_simple.png)

System Design Tutorial Example 7: System design for online messaging service like WhatsApp

So when the user, suppose UserA sends a message to UserB, once mobile is connected to internet, that message will be sent to a load balancer, then to a Message Server.
Then the message server will check if the other user i.e UserB is connected to the service. If UserB is connected, then the message will be sent to UserB. If UserB is not connected to the service, then the message will be stored in DB.
As soon as UserB is connected to the server, then the server will send that message to UserB.

This is how a message will work end to end.

# 2. Understanding feature like last seen, single tick and double tick will work?

These features can be accomplished by Acknowledgement service.

## 2.1. Single Tick:

Once the message from UserA reaches the server, the server will send an acknowledgement saying that the message has been received. Then UserA will display single tick.

## 2.2. Double Tick:

Once the message from server sends that message to UserB by appropriate connection, UserB will send an acknowledgement to the server saying that it has recieved the message.

Then server will send another acknowledgement to UserA, hence it will display double tick.

## 2.3. Blue Tick:

Once the UserB opens whatsapp and checks the message, UserB will send another acknowledgement to server saying that user has read the message. Then server will send another acknowledgement message ti UserA. Then UserA will display blue tick.

To identify all the acknowledgements, there will be unique ID attached to all the message.

# 3. Last Seen feature:

For this, we need a heartbeat mechanism. This service will be sending a heartbeat for every 5 seconds when the user is online or using the application.
When the server receives a heartbeat, it will store in another table with User_name and last seen time.

Then retrieve this information, when UserB is online.

# 4. Working of message server

Whenever a connection is made to the server, a separate thread along with a queue[to store messages] will be created. Then there will be a separate table that will be mapping threadID with DeviceID.

So when the message is received from UserA, Thread of UserA will check the table, if UserB is connected. If UserB is connected, then the message is sent to queue of UserB. Then the thread of UserB will check if there are any messages in its queue. If there are any messages, then those messages will be sent to UserB.

If the UserB is not connected to the service, then the entry of UserB not be there in the table. When UserA sends message to UserB, as the entry is not available, the message will stored in DB. Once UserB is online, then the message will get delivered.

# 5. Understanding how media transfer works?

For sending media, we cannot use previously created thread. As threads are light weight, sending media in the same connection will not be efficient. For this you can use a HTTP connection to upload media to a http server. Then this server will return a HashID to UserA. Then we send the HashID along with media type to UserB.

When the message is recieved to UserB, UserB will download from http server. This procedure will work for all the media types.

# 6. For encryption, you can use

1. One key that is shared between 2 clients.
2. One user will have a private key and share public key to other user.

But, what if the network is very large and number of clients are in millions or billions?
In a very large network it is very difficult to know the address of each and every client, In this case to make this system more robust and highly available we need a component between the clients called “SERVER”. The task of this server is to coordinate between all the clients connected to it.


After the introduction of server. All clients instead of making connection with each other they make connection with the server.
In this scenario when a client(A) want to send message to other client(D), it first sends the message to server and the server knows the address of other client(D),then it forwards the message to other client(D) and vice versa.
This is the overview of the communication architecture. Lets design Actual System design of a real time messaging system.
But before designing any product it is really important to understand the some requirements like:

1. User base: It is really important to understand at what scale the application is to be used.
2. Features required.

So, let’s list some of the features needed to be incorporated in whatsapp:
1).Text messaging.
2). Media Support.
3). Last Seen
4). Message Encryption
5). Telephony services(Audio/Video)

**Let’s start designing the system based on the application requirement.**

Based upon the user base we need multiple servers to handle this much traffic, so instead of one server we place multiple servers.
But the question is that, to which server the client will connect as there are multiple servers and the client cannot connect randomly to any server. To overcome this issue we introduced a load balancer between the client and the server.

After implementing multiple servers and load balancer, our system architecture is capable of handling a large user base. Now when a client want to connect to the server, the connection request first hits the load balancer and then the load balancer redirects the connection to a server based on various parameters like load on individual servers etc.


![alt text](https://miro.medium.com/max/1110/1*is2GaFuYvO_d9SLLpNDJvg.png)


But our application also needs some storage mechanism to save some arbitrary state or data, to fulfill this requirement we also added database which is accessible to all the servers.
But, What kind of connection is used?
Generally, this kind of system uses a DUPLEX Connection or Bidirectional Connection. As the message can also be generated from the server, so bidirectional communication is required
Before moving ahead lets understand different connectivity scenarios and how the application works.

1. When Sender is connected to server but not the receiver.

In this case when the receiver is not connected to the server, the message is stored in the database and when the receiver connects to the server, the message is fetched from the database and forwarded to the receiver.

2 . When Sender is not Connected to the Server.

In this case when sender is not connected to the server, the message sent by the sender is saved in the device local storage (it may be SQLite or anything else based on platform). And when the sender goes online or connects to the serve the message is fetched from the local storage and sent to the server.

![alt text](https://miro.medium.com/max/1086/1*7JkjITVEEoUlUAC5USUgkA.png)

3 . When both clients are connected to the server:

In this case when both the clients are connected to the server, the sender sends the message and the server forwards that message to the receiver without storing the message to the database or device local storage.

One thing which needs to be known is that the connection is always initiated by the client because the server does not know the address of the client but the client knows the address of the server.

**How Sent, Delivered and Seen Works?**

To Incorporate all these status changes, every message has a unique ID to identify each message and acknowledgement from the various events (sent / delivered /seen).

**What happens inside the Whatsapp server when a client connects to the server?**

![alt text](https://miro.medium.com/max/770/1*KIo2e5Op_iUb4j-6z_SkKQ.png)

When a client connects to the WhatsApp server, a process (or thread) is created with respect to that client. This process is responsible for handling all the operations related to that client.
With every process, a queue(Highlighted with light green colour) is associated which act as a buffer for that process. After process creation, a table is created in the database to maintain the record of PID(Process ID) and the associated Client.

**How Last Seen Work?**
Implementation of this feature is very simple and straightforward, It is just about maintaining a record with Client ID and Timestamp.

![alt text](https://miro.medium.com/max/730/1*KWR0NAiGqnU-HzuFP-Zurg.png)

When we open Whatsapp in our smartphone, our application sends a pulse to server every 5 seconds, and with every pulse last seen time is updated in the table. As the client disconnects the last seen time exists in the record that is updated by the last pulse sent before closing the app.

**How the media sharing works?**

For sharing, we don’t use the connection which is used for sending text messages because it is a very lightweight connection and it cannot handle this much load.
Instead, WhatsApp uses a different server(like HTTP) to share media.

![alt text](https://miro.medium.com/max/770/1*pkKmT7Rtl1l9DKfZQAB_nA.png)


When we share a media, it gets uploaded to an HTTP Server over a different connection, after successful upload, the HTTP server returns a hash or unique ID associated to that media and that hash value is sent to the WhatsApp server. At the receiver end, the same thing works in a reverse way, the receiver receives the hash value then it downloads the media from HTTP server associated to that hash value.

The Telephony services also work in the same way just like media services, for this, we also use a different server and use a different kind of connection like socket etc. for real-time communication.
This is all about the overview of a real-time messaging system.

Let's Talk about the actually Technology used by Whatsapp :)
-> Programming Language: Erlang

Erlang is a super fast programming language which supports features like Hot Reload/Update on Fly etc. It also has a concept of the lightweight thread which makes it capable of handling millions of connections at a time. This is the reason Erlang is an ideal choice for WhatsApp.

In Actual, Whatsapp handles 10 million connection on a single server, which seems to be impossible but the WhatsApp team able to achieve this. And it is only possible if you know all the things about the system like Server kernel, networking library, infrastructure configuration etc.

-> Operating System on Servers: FreeBSD is the OS used by all the messaging servers of WhatsApp. As it is open source OS and the developer knows all the in and out. so that they can get maximum performance out of it.
-> Database: AMNESIA is the database which is used for storing data, it is also a key-value pair based DB which couples really good with Erlang.
-> Web Server: The Web Server used in all the messaging server is YAWS — Yet Another Web Server.

# How WhatsApp is different from other messaging services?

In WhatsApp, the text is end to end encrypted. Hence intermediate servers cannot read your messages.
In WhatsApp, the messages are stored in server till the other party reads the message. Once other party reads the message, that message will be deleted from the server.

