---
title: "What About Transactions?"
date: 2019-06-12T13:00:00+01:00
draft: false

categories: ["wcf"]
tags: ["microservices", "transactions"]
author: "Mark Rendle"
---

A few people have asked about the alternative to WCF's distributed transaction support, as
implemented with `[TransactionFlow]` attributes and the .NET `TransactionScope` type. That
built-in support for complex transaction management was one of the best features of WCF, but
it's also one of the reasons why a wholesale port to .NET Core is so difficult: under the covers,
WCF's transactions were managed by the
[Microsoft Distributed Transaction Coordinator](https://en.wikipedia.org/wiki/Microsoft_Distributed_Transaction_Coordinator) 
(MSDTC), which is a Windows service and is integrated with Microsoft's (and other vendors') software
when that software is running on Windows.

Modern distributed systems can include multiple platforms, frameworks, programming languages,
hosting environments and even multiple data-centres and clouds. That makes creating a high-level
distributed transaction coordinator that can cope with all those partitions next to impossible
(I would love to be corrected on that statement, BTW).

Given this reality, there are three main options available when designing or building distributed applications:

1. Avoid using distributed transactions altogether.
2. The 2-Phase Commit pattern.
3. The Saga pattern.

#### Avoid using distributed transactions

While this is not always an option, you may find that where you used some form of atomic transaction
simply because doing so was easy, it's not actually necessary. For example, you may keep a count of
"likes" for a post as a `LikeCount` column on the `Posts` table, as well as an actual record of each "like"
with the user details and so on in a separate `Likes` table. In a case like this, the update to the
`LikeCount` column could be handled asynchronously via a queue or message bus with retry semantics, or
you could try to query a `Likes` microservice and use the `LikeCount` column as a sort of fallback cache.

Alternatively, you could use the fact that a particular set of modifications to data *must* be handled atomically
as a form of service boundary, and keep that data together in a store that supports ACID transactions. Once the
update has completed, copies of the committed data could be passed on to other services in the background.
Of course, this approach introduces constraints on your architecture that may not be possible or desirable.

#### 2-Phase Commit pattern

With the [2-Phase Commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) (2PC) pattern,
you build the transaction awareness into each service that participates in
the transaction, and also build something that acts as the coordinator; that might be the client, or an
API endpoint that distributes work across multiple backend services, or a dedicated coordinator service.
Either way, the coordinator sends a request to each service, and they begin a transaction and make the update,
then send a response to the coordinator confirming that they are ready to commit. If all the services respond
positively, a second request is sent to commit the transactions. If any service responds negatively, or fails
to respond in time, the second request instead tells the services to rollback the transactions. This is effectively
what WCF transactions have been doing, using Windows' MSDTC as the coordinator.

gRPC's streaming requests and responses are particularly helpful in implementing 2PC, because they can provide a
stateful connection between the coordinator and the services. With a bi-directional streaming endpoint, the
coordinator can start a call, send a Begin message, receive a Ready-to-Commit message, and then send a Commit
message and receive a Finished message, and this can be handled across multiple services concurrently.

There are a couple of issues with 2PC transactions in practice. Firstly, it's almost impossible to actually
*guarantee* the success of the transaction across multiple services; having responded with a success status,
a service might crash before receiving the commit request, or the commit action itself might fail. Secondly,
2PC requires multiple services to hold open connections and transactions against the underlying database or
other store, which causes problems with both performance and scalability of the overall system; it also means
that 2PC is not at all suited to long-running transactions.

#### Saga pattern

The [Saga pattern](https://microservices.io/patterns/data/saga.html)
is the most complicated form of distributed transaction to implement, but also the most robust.
Rather than being executed all at once, the different steps are handled sequentially by various services,
with each service reporting the success or failure of its step. In the event that all services report success,
the overall operation is also reported as successful. If a single service reports failure, the operation stops
and additional requests are sent to any previously successful services telling them to undo the changes; for
example, to refund money to a customer's account, or to increment the stock count of a previously decremented
item.

Because there are no held connections or transactions against databases, the Saga pattern is far more scalable
than the 2PC approach, and can handle operations that take a long time to complete. But with this comes the
possibility of data staleness or inconsistency across services, and you have to design everything with that
possibility in mind.

As you can no doubt tell, building systems that use the Saga pattern is complicated, but a lot of what is involved
is also just good practice, and has a lot in common with
[Domain Driven Design, CQRS](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/),
and [Event Sourcing](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing). These patterns
have become popular for good reason, and should be a part of every good architect's thinking about modern
system design, distributed transactions or no.

#### No magic bullet

In summary, then, the unhappy truth is that there is no simple, drop-in replacement for WCF transactions if you are migrating
to modern microservice architectures, whether with gRPC, HTTP APIs or any other protocols. If the engineering challenges
involved in migrating those parts of your system that absolutely require ACID transactional integrity are too great
for your organisation to take on right now, then it may be that you have to keep those parts running on full-blown,
Windows-only, .NET 4.8 WCF for the time being. But that doesn't mean that you can't start to separate out the other
parts of the system, the parts that don't need distributed transactions, and migrate them to .NET Core, and run them on
Linux servers or containers or whatever. At the very least, once you've moved all the parts that are easy to move,
what remains will be easier to reason about and maybe the path ahead will be clearer.
