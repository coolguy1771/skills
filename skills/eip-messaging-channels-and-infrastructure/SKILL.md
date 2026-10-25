---
name: eip-messaging-channels-and-infrastructure
description: Apply the Enterprise Integration Patterns (Hohpe & Woolf) for messaging infrastructure — Message, Message Channel, Point-to-Point Channel, Publish-Subscribe Channel, Datatype Channel, Invalid Message Channel, Dead Letter Channel, Guaranteed Delivery, Channel Adapter, Messaging Bridge, Message Bus, Pipes and Filters, Message Router, Message Translator, and Message Endpoint. Use this whenever the user is choosing between queues and topics, designing channel topology for Kafka/RabbitMQ/Service Bus/SQS/SNS/JMS, asking "queue or topic," "one channel per message type or one shared channel," "what happens to messages that can't be delivered or parsed," "how do I connect a legacy app to the broker," "how do I connect two different brokers," or "how do I make sure messages survive a broker crash." Also use it to review an event-driven architecture's channel design.
---

# EIP: Messaging Channels and Infrastructure

These are the root patterns of *Enterprise Integration Patterns*. Every other EIP pattern (routing,
transformation, endpoints, management) assumes them. Settle channel design first. A badly chosen
channel topology can't be fixed later with clever routers.

## The core model

| Pattern | Problem it answers | Distilled rule |
|---|---|---|
| [Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Message.html) | How do two apps connected by a channel exchange information? | Package data as a **header** (metadata the messaging system uses) plus a **body** (data the app uses). Keep routing, correlation, and versioning info in headers so infrastructure never has to parse the body. |
| [Message Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageChannel.html) | How does one app talk to another through messaging? | Apps write to and read from a named **logical address**, never directly to each other. Channels are fixed when you design the system. Treat their names as a public API. |
| [Pipes and Filters](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html) | How do you do complex processing while keeping steps independent? | Break processing into **filters** with one input and one output, joined by **pipes** (channels). Each filter can be swapped, reordered, scaled, or tested on its own. This is the shape of Unix pipelines, Camel routes, and stream topologies. |
| [Message Router](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageRouter.html) | How do you send messages to different filters depending on conditions? | Insert a filter that **consumes from one channel and republishes to one of several** without changing the content. Producers stay unaware of where messages end up. Variants are in `eip-message-routing`. |
| [Message Translator](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageTranslator.html) | How do systems with different data formats communicate? | Put a translator filter between them, the messaging equivalent of the Adapter pattern. Translation happens at several layers: transport, data types, data structures (entities and cardinality), and representation (XML vs JSON). Name the layer you're changing. Variants are in `eip-message-transformation`. |
| [Message Endpoint](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageEndpoint.html) | How does an app connect to a channel? | Wrap broker client code in an endpoint so the rest of the app doesn't know messaging exists. An endpoint either sends or receives, not both. Variants are in `eip-messaging-endpoints-and-consumers`. |

## Choosing the channel type

| Pattern | Use when | Watch out for |
|---|---|---|
| [Point-to-Point Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PointToPointChannel.html) | **Exactly one** receiver must process each message: commands, work items, a call to a single provider. | Several receivers on one point-to-point channel *compete*, and each message goes to only one of them. That's how you scale out (Competing Consumers), but it surprises people who expected every receiver to get a copy. Queues: SQS, RabbitMQ queue, Service Bus queue, JMS Queue. |
| [Publish-Subscribe Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html) | **Every** interested receiver must get a copy, as with event notifications. | One input channel splits into one output channel per subscriber, and each subscriber consumes its copy once. Subscribers only receive messages published while they're subscribed unless you add Durable Subscriber. Topics: SNS, Kafka topic + consumer groups, Service Bus topics, RabbitMQ fanout exchange. |
| [Datatype Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DatatypeChannel.html) | The receiver must know how to process the data without inspecting it. | Use **one channel per data type**, so everything on a channel has the same type. Better than a shared "everything" channel plus type sniffing. The cost is more channels. Format Indicator handles versions of the same type. |

**Heuristic:** commands go point-to-point. Events go publish-subscribe. Don't mix data types on one
channel unless every consumer genuinely handles every type.

## When things go wrong

| Pattern | Distilled rule |
|---|---|
| [Invalid Message Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/InvalidMessageChannel.html) | When a receiver **gets a message it can't make sense of** (bad format, schema violation, semantically impossible), it moves the message to a dedicated invalid-message channel. It doesn't crash, loop, or silently drop it. This is an *application-level* decision. |
| [Dead Letter Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DeadLetterChannel.html) | When the **messaging system can't deliver** a message (expired, too many retries, unknown destination), it moves it to a dead letter channel. This is a *system-level* decision. Both channels need an **owner**: alerting, a triage runbook, and a replay path. A DLQ nobody watches loses messages slowly. |
| [Guaranteed Delivery](https://www.enterpriseintegrationpatterns.com/patterns/messaging/GuaranteedMessaging.html) | Make messages **persistent**: store-and-forward to disk on sender, broker, and receiver, so a crash doesn't lose them. Costs throughput and disk. In practice this means durable queues + persistent messages + publisher confirms (RabbitMQ), `acks=all` + replication (Kafka), or similar. Guaranteed delivery gives you *at-least-once*, so pair it with Idempotent Receiver. |

Keep the two failure channels separate. Invalid-message triage is a schema/contract problem owned
by the producer. Dead-letter triage is an operational problem (capacity, availability, TTL).

## Connecting things to the messaging system

| Pattern | Distilled rule |
|---|---|
| [Channel Adapter](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ChannelAdapter.html) | For apps that weren't built for messaging (legacy, packaged, a database), an adapter calls the app's API, UI, or database and publishes and consumes messages for it. Kafka Connect, Debezium, and Logic Apps connectors are channel adapters. Decide whether the adapter works at the **UI**, **business logic/API**, or **database** layer. Database-level is easiest but couples you to the schema. |
| [Messaging Bridge](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessagingBridge.html) | To connect **two messaging systems** (e.g., on-prem MQ to cloud Service Bus), a bridge consumes from a channel on one and republishes on the matching channel on the other, one channel pair at a time. Examples: MirrorMaker, shovels, federation. Watch for duplicates and ordering across the hop. |
| [Message Bus](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageBus.html) | To let apps join and leave without affecting each other, combine three things: a **shared messaging infrastructure**, a **canonical data model**, and a **common command set** (agreed channels and message types). The bus is the agreement as much as the technology. Without the canonical model you only have shared plumbing. |

## Review checklist for a channel design

1. Every channel has a single data type (or an explicit Format Indicator) and an owner.
2. Each channel is explicitly point-to-point or pub-sub, matching command vs event semantics.
3. Separate invalid-message and dead-letter channels exist, are monitored, and have a replay procedure.
4. Persistence is on where losing a message is unacceptable, and consumers are idempotent.
5. Legacy systems connect through adapters, not by having the broker reach into them.
6. Routing info is in headers, so routers never parse bodies.

## Related skills

- `eip-message-construction`: what goes *in* the message (command, event, document, correlation)
- `eip-message-routing` and `eip-message-transformation`: the filters between channels
- `eip-messaging-endpoints-and-consumers`: how apps attach to channels
- `eip-system-management`: observing and controlling the channels
- `integration-patterns-selection-guide`: map a problem to the right pattern family
