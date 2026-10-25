---
name: eip-message-construction
description: Design message intent and metadata using the Enterprise Integration Patterns for message construction — Command Message, Document Message, Event Message, Request-Reply, Return Address, Correlation Identifier, Message Sequence, Message Expiration, and Format Indicator. Use this whenever the user asks "should this be a command or an event," "how do I do request/response over a queue," "how does the reply find its way back," "how do I match replies to requests," "how do I send a payload too large for one message," "how do I stop stale messages being processed," "how do I version my message schema," or is defining message envelopes, headers, CloudEvents attributes, or async API contracts.
---

# EIP: Message Construction

The channel carries the message. These patterns decide **what the message means** and **which
metadata it needs** for the receiver to act on it correctly. Most hard-to-debug messaging bugs
(orphaned replies, a stale command executed twice, a schema change that breaks every consumer)
come from metadata someone left out here.

## Step 1: name the message's intent

| Pattern | Intent | Sender expects | Typical channel |
|---|---|---|---|
| [Command Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CommandMessage.html) | "Do this." Invokes a procedure in another app (an asynchronous RPC). | Exactly one receiver to perform it, often with a reply | Point-to-Point |
| [Document Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DocumentMessage.html) | "Here is data." Transfers a data structure. The receiver decides what to do with it. | Data delivered reliably; timing usually matters less | Point-to-Point (or pub-sub for broadcast) |
| [Event Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/EventMessage.html) | "This happened." Notifies others of a change. | Nothing. Fire and forget, any number of observers | Publish-Subscribe |

Distilled rules:
- **Name commands in the imperative** (`ReserveInventory`) and **events in the past tense**
  (`InventoryReserved`). If you can't tell from the name which one it is, the design is muddled.
- Events have two styles. **Push** puts the changed state in the event (event-carried state
  transfer). **Pull** sends a thin notification and the observer fetches the details. Push decouples
  in time but makes events bigger and harder to version. Pull keeps events small but couples
  observers to a query API and opens a race between the notification and the fetch.
- Events are timeliness-sensitive. Documents are content-sensitive. Commands are both. This decides
  whether Message Expiration and Guaranteed Delivery matter.

## Step 2: if you need a reply

| Pattern | Distilled rule |
|---|---|
| [Request-Reply](https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReply.html) | Use **two channels**: the requestor sends on a request channel and listens on a reply channel. The reply channel is usually point-to-point. The requestor either blocks (synchronous over async) or registers a callback. A broadcast request needs Scatter-Gather (`eip-message-routing`). |
| [Return Address](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ReturnAddress.html) | The request carries **the address of the reply channel** (JMS `JMSReplyTo`, AMQP `reply_to`, Service Bus `ReplyTo`). The replier never hard-codes where replies go, so one service can answer many requestors, and requestors can use temporary or per-instance reply queues. |
| [Correlation Identifier](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html) | Each request gets a unique ID. **The reply copies it into a correlation-ID header**, so the requestor can match replies to outstanding requests even when they arrive out of order or several are in flight. Usually you correlate on the request's *message ID*. Use the same mechanism for tracing (e.g., a W3C `traceparent` propagated end to end). |

Always use Return Address and Correlation Identifier together. A reply with no correlation ID is an
orphan. Also keep a timeout per outstanding request, so a reply that never arrives doesn't leak
memory.

## Step 3: if the payload is too big or arrives in parts

[Message Sequence](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageSequence.html)
splits large data into numbered parts. Each part carries **three fields**: a sequence ID shared by
all parts, a position, and either a total count or an end-of-sequence flag. The receiver
reassembles once all parts arrive (often with a Resequencer + Aggregator).
- Before using it, ask whether a **Claim Check** (store the payload, send a reference) is simpler.
  Usually it is, now that blob storage is cheap.
- Decide what happens when a part goes missing: time out and discard, or request a resend. Partial
  sequences must not be processed as complete.

## Step 4: time and versioning metadata

| Pattern | Distilled rule |
|---|---|
| [Message Expiration](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageExpiration.html) | Set an **absolute expiry time or a time-to-live** on messages that are useless after a point: price quotes, real-time commands, a "start the car" request. The broker drops or dead-letters expired messages. Receivers should *also* check expiry, because clocks and redeliveries can bypass broker enforcement. Use absolute timestamps (UTC) when processing crosses hops. A TTL resets per hop on some brokers. |
| [Format Indicator](https://www.enterpriseintegrationpatterns.com/patterns/messaging/FormatIndicator.html) | Put an explicit format marker in the message so the format can change later: a **version number**, a **foreign key to a format document** (schema registry ID, `dataschema` URI), or a **format document embedded** in the message. Add it to the first schema you publish. Retrofitting a version field after the first breaking change is painful. Confluent/Apicurio schema IDs and the CloudEvents `dataschema` attribute are modern versions of this. |

## Message metadata checklist

For each message type, confirm:

1. **Intent** is clear from the name: command, document, or event.
2. **Message ID** is unique per message (needed for idempotency and correlation).
3. **Correlation ID** and **reply-to** are present when a reply is expected.
4. **Timestamp** (UTC) and **expiry** are set if the message is time-sensitive.
5. **Format/version indicator** is present.
6. **Sequence fields** (ID, position, count/end flag) are present if the message is part of a set.
7. **Trace context** is propagated for observability.

CloudEvents (`id`, `source`, `type`, `time`, `dataschema`, `datacontenttype`, plus extensions) is a
ready-made envelope that covers most of this checklist. Prefer it over a home-grown header scheme
when the platform supports it.

## Related skills

- `eip-messaging-channels-and-infrastructure`: picking the channel each intent travels on
- `eip-message-routing`: Scatter-Gather, Aggregator, and Resequencer for multi-part and multi-reply flows
- `eip-message-transformation`: Claim Check and Envelope Wrapper
- `map-api-evolution-and-versioning`: versioning strategy for the formats you indicate
