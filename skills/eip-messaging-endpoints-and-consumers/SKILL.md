---
name: eip-messaging-endpoints-and-consumers
description: Design how application code connects to messaging using the Enterprise Integration Patterns for endpoints — Messaging Gateway, Messaging Mapper, Transactional Client, Polling Consumer, Event-Driven Consumer, Competing Consumers, Message Dispatcher, Selective Consumer, Durable Subscriber, Idempotent Receiver, and Service Activator. Use this whenever the user asks how to hide broker APIs from domain code, push vs pull consumers, how to scale consumers horizontally, how to handle duplicate messages or at-least-once delivery, exactly-once processing, the transactional outbox, "consumer missed messages while it was down," filtering on the consumer side, or exposing the same service over HTTP and a queue. Also trigger for Kafka consumer group design, SQS/Service Bus/RabbitMQ consumer code review, and Lambda/Functions queue triggers.
---

# EIP: Messaging Endpoints and Consumers

Endpoints are where messaging meets application code, and where most production defects show up:
duplicates processed twice, a message lost when a consumer crashed mid-transaction, a slow
consumer blocking the queue. Get these patterns right and the rest of the integration is far more
forgiving.

## Keep messaging out of the domain

| Pattern | Distilled rule |
|---|---|
| [Messaging Gateway](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingGateway.html) | Put all messaging access behind a **domain-specific interface**. Domain code calls `orderService.submit(order)`, not `producer.send(...)`. The gateway can be synchronous (it hides request-reply) or async (it returns a future or callback). It makes broker swaps and test doubles trivial. |
| [Messaging Mapper](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingMapper.html) | A separate mapper converts **domain objects to messages and back**, so neither the domain model nor the messaging layer depends on the other. Like a data mapper for an ORM. It also handles references, e.g., by sending IDs instead of whole object graphs. |
| [Service Activator](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingAdapter.html) | Write the service as a **plain synchronous interface**, then add a thin activator that receives messages, calls the service, and sends any reply. The same service can then be invoked over HTTP, gRPC, or a queue. Lambda/Functions queue triggers are service activators. |

## Push or pull?

| Pattern | How it works | Choose it when |
|---|---|---|
| [Polling Consumer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PollingConsumer.html) | The app **explicitly asks** for the next message when it's ready (a synchronous receiver). | You need **flow control**: the consumer sets its own pace, batches, or only runs on a schedule. Kafka's `poll()` and SQS long-polling work this way. |
| [Event-Driven Consumer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/EventDrivenConsumer.html) | The messaging system **calls the app** when a message arrives (an asynchronous receiver). | You want low latency and simple code, and the platform handles concurrency (JMS listeners, RabbitMQ push consumers, serverless triggers). Set prefetch/concurrency limits, or a burst will overwhelm downstream systems. |

## Scaling consumers

| Pattern | Distilled rule |
|---|---|
| [Competing Consumers](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CompetingConsumers.html) | Several consumers on **one point-to-point channel**. The channel gives each message to only one of them, so you scale throughput by adding consumers. **Cost: you lose global ordering.** If order matters per entity, partition by key (Kafka partitions, Service Bus sessions, SQS FIFO message groups) so order is kept within a key and parallelism comes from the number of keys. |
| [Message Dispatcher](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageDispatcher.html) | **One consumer receives** and **hands messages to performers** (worker threads or handlers) inside the process. You control distribution (round-robin, by type, by key) instead of leaving it to the broker. Useful when the broker's competing-consumer semantics are too coarse or broker connections are expensive. |
| [Selective Consumer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageSelector.html) | The consumer declares a **selector** (a header predicate) and the channel delivers only matching messages. Examples: JMS selectors, Service Bus subscription filters, SNS filter policies. It cuts traffic, but too many selectors on one channel often means it should be split into separate **Datatype Channels**. |

## Reliability

| Pattern | Distilled rule |
|---|---|
| [Durable Subscriber](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DurableSubscription.html) | Make the pub-sub subscription **outlive the subscriber's connection**. The system stores messages while the subscriber is offline. Kafka consumer groups with committed offsets, durable Service Bus subscriptions, and durable RabbitMQ queues bound to an exchange are all durable. Clean up abandoned subscriptions, or they'll pile up messages indefinitely. |
| [Transactional Client](https://www.enterpriseintegrationpatterns.com/patterns/messaging/TransactionalClient.html) | Make the client's send/receive part of a **transaction**: consume plus database write commit or roll back together, or several sends are atomic. Distributed transactions (XA/2PC) are usually unavailable or too slow in cloud brokers. Use the **transactional outbox** instead (write the message to a DB table in the business transaction, and a relay publishes it). On the receive side, use **receive → process → ack-after-commit + Idempotent Receiver**. |
| [Idempotent Receiver](https://www.enterpriseintegrationpatterns.com/patterns/messaging/IdempotentReceiver.html) | At-least-once delivery means **duplicates will happen**. There are two ways to make them harmless. (1) Make the operation semantically idempotent ("set balance to X" or an upsert, rather than "add X"). (2) Record processed **message IDs** in a dedup store and skip ones you've seen, writing the dedup record *in the same transaction* as the side effect. Expire dedup records only after the broker's maximum redelivery window. |

**Exactly-once in practice = at-least-once delivery + idempotent processing.** Treat any claim of
broker-level exactly-once with suspicion once side effects leave the broker: an email sent or a
payment API called.

## Endpoint review checklist

1. Is broker code isolated behind a gateway/mapper? Can the domain logic be unit-tested without a broker?
2. Is the ack/commit sent **after** the side effect is durable, never before?
3. Is the handler idempotent, either by design or with a dedup store written atomically with the effect?
4. Is ordering required? If so, is it scoped to a partition/session key instead of being global?
5. Are concurrency, prefetch, and batch size bounded to what downstream dependencies can handle?
6. Do poison messages reach an Invalid Message/Dead Letter channel after N attempts, instead of
   blocking the partition?
7. Are durable subscriptions inventoried and their lag monitored?

## Related skills

- `eip-messaging-channels-and-infrastructure`: Guaranteed Delivery, Dead Letter Channel
- `eip-message-construction`: message IDs and correlation the receiver relies on
- `eip-system-management`: Message Store and Wire Tap for auditing consumers
- `soa-service-messaging-patterns`: Reliable Messaging and Asynchronous Queuing at the SOA level
