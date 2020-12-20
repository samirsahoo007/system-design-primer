# System Design of Uber/Ola/Lyft App – Uber System Architecture

It’s really easy to just tap a button on our mobile phone and get the cab available within few minutes whenever and wherever we want.
Uber/Ola/Lyft... using these applications and getting the hassle-free transportation service is really simple but is it also simple to build these gigantic applications which have hundreds of software engineers working on it for a decade…? definitely not. These systems have much more complex architecture and there are a lot of components joined together internally to provide riding services all over the world. 

## Uber System Architecture
We all are familiar with Uber services. A user can request a ride through the application and within a few minutes, a driver arrives nearby his/her location to take them to their destination. Earlier Uber was built on the “monolithic” software architecture model. They had a backend service, a frontend service, and a single database. They used Python and its frameworks and SQLAlchemy as the ORM-layer to the database. This architecture was fine for a small number of trips in a few cities but when the service started expanding in other cities Uber team started facing the issue with the application. After the year 2014 Uber team decided to switch to the “service-oriented architecture” and now Uber also handles food delivery and cargo.

### 1. Talk About the Challenges

One of the main tasks in Uber service is to match the rider with cabs which means we need two different services in our architecture i.e.

* Supply Service (for cabs)
* Demand Service (for riders)

Uber has a Dispatch system (Dispatch optimization/DISCO) in its architecture to match supply with demands. This dispatch system uses mobile phones and it takes the responsibility to match the drivers with riders (supply to demand).

### 2. How Dispatch System Works?
DISCO must have these goals...

* Reduce extra driving.
* Minimum waiting time
* Minimum overall ETA

The dispatch system completely works on maps and location data/GPS, so the first thing which is important is to model our maps and location data.

* Earth has a spherical shape so it’s difficult to do summarization and approximation by using latitude and longitude. To solve this problem Uber uses Google S2 library. This library divides the map data into tiny cells (for example 3km) and gives the unique ID to each cell. This is an easy way to spread data in the distributed system and store it easily.

* S2 library gives the coverage for any given shape easily. Suppose you want to figure out all the supplies available within a 3km radius of a city. Using the S2 libraries you can draw a circle of 3km radius and it will filter out all the cells with IDs lies in that particular circle. This way you can easily match the rider to the driver and you can easily find out the number of cars(supply) available in a particular region.

### 3. Supply Service And How it Works?

* In our case cabs are the supply services and it will be tracked by geolocation (latitude and longitude). All the active cabs keep on sending the location to the server once every 4 seconds through a web application firewall and load balancer. The accurate GPS location is sent to the data center through Kafka’s Rest APIs once it passes through the load balancer. Here we use Apache Kafka as the data hub.

* Once the latest location is updated by Kafka it slowly passes through the respective worker notes main memory.
Also a copy of the location (state machine/latest location of cabs) will be sent to the database and to the dispatch optimization to keep the latest location updated.
We also need to track few more things such as number of seats, presence of a car seat for children, type of vehicle, can a wheelchair be fit, and allocation ( for example, a cab may have four seats but two of those are occupied.)

### 4. Demand Service And How it Works?
Demand service receives the request of the cab through web socket and it tracks the GPS location of the user. It also receives a different kind of requirements such as the number of seats, type of car, or pool car.
Demand gives the location (cell ID) and user requirement to supply and make requests for the cabs.

### 5. How Dispatch System Match the Riders to Drivers?
We have discussed that DISCO divides the map into tiny cells with a unique ID. This ID is used as a sharding key in DISCO. When supply receives the request from demand the location gets updated using the cell ID as a shard key. These tiny cells’ responsibilities will be divided into different servers lies in multiple regions (consistent hashing). For example, we can allocate the responsibility of 12 tiny cells to 6 different servers (2 cells for each server) lies in 6 different regions.



