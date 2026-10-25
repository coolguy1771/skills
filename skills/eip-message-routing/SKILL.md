---
name: eip-message-routing
description: Choose and implement Enterprise Integration Patterns for message routing — Content-Based Router, Message Filter, Dynamic Router, Recipient List, Splitter, Aggregator, Resequencer, Composed Message Processor, Scatter-Gather, Routing Slip, Process Manager, and Message Broker. Use this whenever the user asks how to send a message to different consumers based on content, broadcast a request and combine the replies, split a batch into items and reassemble results, restore message order, run a message through a variable sequence of steps, orchestrate a multi-step workflow or saga, or "should this be orchestration or choreography." Also trigger for Apache Camel/Spring Integration/MuleSoft/Logic Apps/Step Functions route design and for reviewing routing logic in event-driven systems.
---

# EIP: Message Routing

Routers decouple *who sends* from *who receives*. They move messages between channels and don't
change the content (that's transformation). The main decision is **how much routing knowledge you
centralize**. A little makes a system flexible. A lot creates a "smart pipe" that owns business
logic and becomes a bottleneck.

## Simple routers: one message in, zero or more out

| Pattern | Problem | Distilled rule | Pitfall |
|---|---|---|---|
| [Content-Based Router](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ContentBasedRouter.html) | One logical function (e.g., inventory check) is implemented by several physical systems | Inspect the content or a header and forward to **exactly one** destination channel | The router has to know every destination. Adding a system means changing the router, so keep the rules in configuration, not code |
| [Message Filter](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Filter.html) | A component shouldn't receive messages it doesn't care about | A router with **one output**: pass the message if it matches, otherwise drop it | Dropping is silent. Log or count what you filter out. Pub-sub + filter trades bandwidth for decoupling |
| [Dynamic Router](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DynamicRouter.html) | A content-based router needs to know every destination | Destinations **register their abilities** with the router on a **control channel**. The router keeps a rules table and routes by it | Registrations that conflict or go stale. The control channel needs the same care as the data channel |
| [Recipient List](https://www.enterpriseintegrationpatterns.com/patterns/messaging/RecipientList.html) | Send to a **dynamically computed set** of recipients | Compute the list (from content, config, or subscriptions) and send a copy to **each** | Unlike pub-sub, the *sender side* controls who gets it. Delivery to the whole list should be atomic, or recoverable when partial |

**Router vs pub-sub + filter.** A router is efficient and gives central control, but it's coupled
to destinations. Pub-sub with consumer-side filters is decoupled, but every consumer sees every
message. Pick based on how often destinations change and how sensitive the data is.

## Composed routers: break apart, process, reassemble

| Pattern | Distilled rule |
|---|---|
| [Splitter](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Sequencer.html) | Split a composite message (an order with line items) into one message per element. Copy into each child the **correlation ID, the sequence position, and the total count** that the Aggregator will need. Also copy parent context the children need, or the children will have to fetch it. |
| [Aggregator](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Aggregator.html) | A **stateful** filter that collects related messages and emits one combined message. Define three things explicitly: the **correlation** key (which messages belong together), the **completeness condition** (wait for all, first best, timeout, or external event), and the **aggregation algorithm** (pick best, concatenate, compute). Aggregator state must survive restarts, so persist it or accept data loss. |
| [Resequencer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Resequencer.html) | Buffer out-of-order messages and release them in sequence-number order. Needs a **bounded buffer** and a policy for gaps (wait, time out, skip). An unbounded resequencer waiting on a lost message is a memory leak. Kafka partition keys often remove the need for one. |
| [Composed Message Processor](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DistributionAggregate.html) | **Splitter → Content-Based Router (one route per element type) → Aggregator.** Use it when the elements of one message need different processing and the result must come back as a single message. |
| [Scatter-Gather](https://www.enterpriseintegrationpatterns.com/patterns/messaging/BroadcastAggregate.html) | **Broadcast (pub-sub or Recipient List) → many repliers → Aggregator.** For example, request quotes from several vendors and keep the best one. Set the completeness condition first: a timeout plus a minimum reply count is the usual choice. |

Splitter vs Scatter-Gather: Splitter sends *different parts* to processors. Scatter-Gather sends
*the same request* to several processors.

## Routing through many steps

| Pattern | Distilled rule | Choose it when |
|---|---|---|
| [Routing Slip](https://www.enterpriseintegrationpatterns.com/patterns/messaging/RoutingTable.html) | Attach an **itinerary** (an ordered list of steps) to the message at the start. Each step does its work, removes itself, and forwards to the next. No central coordinator. | The step sequence differs per message but is **known up front** and **linear**, e.g., a validation chain chosen by customer type. |
| [Process Manager](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html) | A central component **keeps state for each process instance** and decides the next step after each result comes back. It can branch, loop, run steps in parallel, and compensate. | The steps **depend on intermediate results**, aren't sequential, or need compensation (sagas). Orchestration engines (Temporal, Step Functions, Camunda, Durable Functions) are Process Managers. |
| [Message Broker](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageBroker.html) | A **central hub** receives messages from many sources, decides the destination, and forwards. Hub-and-spoke, which simplifies the N×M connections between apps. | You need central control and visibility over flows. Don't let it become a single point of failure or a dumping ground for business logic. Federate or partition the hub as it grows. |

**Orchestration vs choreography.** A Process Manager is orchestration: easy to see and change the
flow, but it couples participants to the coordinator. Event-driven choreography plus routing slips
decentralizes the flow, but the overall process becomes implicit and hard to trace. A common
default: choreography inside a bounded context, orchestration for long-running cross-context
business processes with compensation.

## Design checklist

1. Each router's rules live in versioned configuration, and every message that matches no rule
   goes to an Invalid Message or Dead Letter Channel instead of disappearing.
2. Splitters stamp correlation ID + position + count. Aggregators define correlation,
   completeness, and timeout.
3. Stateful components (Aggregator, Resequencer, Process Manager) persist state and bound their
   buffers.
4. Business decisions stay in services. Routers only decide *where*, never *what the business does*.

## Related skills

- `eip-message-construction`: Correlation Identifier and Message Sequence fields that routers depend on
- `eip-system-management`: Detour, Wire Tap, and Message History for observing routes
- `soa-service-composition-patterns`: SOA-level orchestration and compensating transactions
