# Event-sourcing into working memory to improve data access latency

In this article, we describe a modernization, moving from a database-centric access pattern, towards event-sourcing data changes directly into the application's working memory (RAM). 

## Modernization scenario

### Introduction and a naïve starting point

The modernization started with a 'traditional' web application, which leverages both internal data, as well as calling into external APIs. In our scenario, the internal data represents configuration information necessary for the business logic in the app to handle the responses from the external APIs. An example could be an e-commerce site, which calls various external catalogues for goods, and applies business rules to the aggregated result. The internal configuration data in this scenario could e.g. be markup rules or the list of external APIs which can be called. When the business onboards a new catalogue provider, the admin team would update the internal configuration database with the new catalogue configuration. 

During request handling, the web app needs to query the internal configuration database to retrieve the current list of external dependencies such as catalogues, business rule configurations, etc.. The configuration information can change during normal operations, so the application developer has to work on a strategy to determine how often the underlying config database should be queried. In the worst case, the web application must query the configuration database multiple times, for example to determine the different external providers, as well as to retrieve per-provider business rules. 

Querying the configuration source on a per-request basis ensures to always work on the most recent configuration, but adds considerable load to the configuration database, increases the end-to-end latency of each request and therefore reduces the overall capacity of a web application node.

The diagram below describes our starting point, with the client sending requests to the load balancer (a), the load balancer distributing incoming requests to the application servers running the web application (b), and the web application querying both the configuration source (c), as well as the external dependencies (d). The *configuration source* abstractly represents the actor that changes the configuration, which could be business administrators, or automated processes.

![01-Regular-SQL](2023-01-25--event-sourcing-1_01-Regular-SQL.svg)

### Caching in the ORM

Accessing a relational database like our configuration database often is implemented using an object relational mapper (ORM) technology, such as .NET Entity Framework (EF). ORMs support caching query results in memory, so that the amount of database  queries can be significantly reduced. 

In the diagram below, we represent the ORM as a 'local cache' inside the web application:

![02-Cached-SQL](2023-01-25--event-sourcing-1_02-Cached-SQL.svg)

#### Going to extremes - Caching the whole database

Some customers choose a rather aggressive approach to caching: When their application servers start, they 'pre-warm' the node by **completely** querying all tables in the configuration database, to pull the complete configuration information into the application's working memory. 

The advantage of that approach is that it completely removes the necessity to query the configuration database during the request/response lifecycle, because all configuration information is cached locally. 

Unfortunately, this approach brings significant downsides: 

- **Application start times:** Application start takes a considerable amount of time, because the pre-warming process needs to download the full database, which can take quite some time. During that phase, the application is not ready to serve incoming requests. This can be a problem in scale-out situations: imagine an unforeseen spike in the number of incoming requests, for example due to a TV commercial. When a scaling logic increases the number of web servers, it would take quite some time until these additional servers can help handling traffic; the spike might actually already have created problems on the other nodes.
- **Database load:** You must also consider the load patterns on the database: During such a scale-out event, a potentially large number of servers will intensely query the database at the same time, potentially overloading the database. So a 'trampling herd' of customers in the web tier, leading to a scale-out action, might lead to a trampling herd of web servers bringing the database down.
- **Configuration updates:** Runtime updates to the configuration database are not reaching the web application any longer: When everything is cached in RAM, and the ORM no longer queries the database, it might not pull updated data from the configuration database into the application. In the past, we have seen customers who "solved" this problem by rebooting all web servers, one after the other, thus forcing each web server to re-download the entire database *again*, with the updated configuration.

### Introducing Event Sourcing into the architecture

In this web application, the configuration information should be completely kept in the working set, but we want to avoid the aforementioned disadvantages: 

The application should ...

1. ... start quickly (load the configuration data into RAM), 
2. ... do so without bringing down the configuration system, and 
3. ... configuration changes should be visible as fast as possible (without having to restart the application).

The following approach will help with the second and third requirement (we handle quick startup in the next step):

#### Event sourcing

> In this article, I use the term 'event sourcing' quite liberal: Simply speaking, all configuration change events in the system change a certain part of the overall state. 

Let's introduce event sourcing by using the simple analogy of a bank account: When a customer opens an account (which is the very first event), the account has a zero balance. When the customer receives money (a second event), the account balance is increased by the given amount. When the customer wires money to a friend, this third event in the ledger results in having a lower account balance again. The different events (account creation, credit and debit transactions, account closure) represent the banking-specific (domain) events. The account's current balance represents the 'state' of the account. Replaying (sourcing) all the events from the beginning, in the order in which they appeared, always brings leads to the same result. 

#### An "append-only log" data structure and service

As first step in refactoring the architecture, we 'replace' the configuration database with an 'append-only log' data structure. 

> The term *'log'* does not refer to log file entries (like an HTTP request log), but should be interpreted like the "captain's log" on a ship, in which all important events are written down sequentially, and historic records (the past) is not modified. 

'Append-only' means that previously written events are not modified by newly arriving events, instead new events are appended at the end of the log structure. Given that this append-only log must be read by all the web application servers, it must be centrally hosted by a service. Typical implementations of such a service include Apache Kafka, RabbitMQ Streams, or Azure Event Hubs. 

> Such an event stream could loosely be compared to a database's transaction log, in which all state changes to the various database tables are recorded sequentially as well. Replaying the transaction log allows the reconstruction of the database state, like in event sourcing.

The illustration below demonstrates the concept: The configuration source emits state update events into Event Hub, and all running web app nodes receive (pull) their individual copy of these changes. When sending (enqueueing, appending) new messages into Event Hub, each message get's a unique **sequence number** assigned, a strictly monotonic increasing integer that uniquely identifies the message. 



![03-EventSourcing-Pure](2023-01-25--event-sourcing-1_03-EventSourcing-Pure.svg)

#### Pulling events into the application

In our updated architecture above, we augment the application with an active component, that pulls a copy of the stream from the append-only log structure (Azure Event Hubs service), and locally applies these events / updates / deltas to the local copy of the configuration state. 

Each received event locally updates the cached state within the application, so that a new 



EventSourcing-with

![04-EventSourcing-with-Snapshots](2023-01-25--event-sourcing-1_04-EventSourcing-with-Snapshots.svg)





DataPump](

![05-DataPump](2023-01-25--event-sourcing-1_05-DataPump.svg)





EventSourcing-Start

![07-EventSourcing-Start](2023-01-25--event-sourcing-1_07-EventSourcing-Start.svg)





EventSourcing-ResumeHot

![08-EventSourcing-ResumeHot](2023-01-25--event-sourcing-1_08-EventSourcing-ResumeHot.svg)





09-EventSourcing

![09-EventSourcing-ResumeFromCapture](2023-01-25--event-sourcing-1_09-EventSourcing-ResumeFromCapture.svg)

## Links

- [Event Sourcing pattern - Azure Architecture Center | Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- 
