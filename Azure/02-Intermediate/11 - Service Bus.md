---
tags: [azure, intermediate, messaging]
aliases: [Service Bus, ASB, Topics, Queues]
level: Intermediate
---

# Service Bus

> **One-liner**: **Azure Service Bus** is an enterprise message broker — **queues** for point-to-point, **topics + subscriptions** for pub/sub, with **sessions** (FIFO + state), **dead-letter queues**, **scheduled delivery**, **duplicate detection**, and at-least-once + transactional semantics.

---

## Quick Reference

| Concept | Meaning |
| ------- | ------- |
| **Namespace** | The Service Bus resource; contains queues + topics |
| **Queue** | Single producer/consumer pattern; FIFO per session |
| **Topic** | Pub/sub source |
| **Subscription** | Filtered virtual queue under a topic |
| **Message** | Body + properties + system metadata |
| **Session** | Subset of messages with the same SessionId; serializes a sender to a consumer |
| **DLQ** | Dead-letter sub-queue for poison/timeout messages |
| **Lock** | Receiver holds a peek-lock; renew or complete before timeout |
| **TTL** | Message expires; goes to DLQ if `DeadLetteringOnMessageExpiration=true` |
| **Auto-forward** | Chain queues/topics — output of A becomes input of B |

| Tier | Highlights |
| ---- | ---------- |
| **Basic** | Queues only; no topics; very limited |
| **Standard** | Queues + topics; most workloads |
| **Premium** | Predictable performance, geo-DR, larger messages (100 MB), private endpoints |

---

## Core Concept

A **queue** decouples sender and receiver in time. The sender writes; the receiver pulls when ready. Service Bus guarantees **at-least-once** delivery (so consumers must be idempotent) and **FIFO within a session**.

A **topic** is a queue with multiple **subscriptions**, each with its own filter rules. Publish once; subscribers each see a copy. Used for fan-out (e.g., one `OrderPlaced` event → `Billing`, `Inventory`, `Notification` subs).

**Peek-lock** is the receive mode: the broker hands you a message + lock; you process; you `Complete` (delete) or `Abandon` (release) or `DeadLetter` (move to DLQ). Forget to complete? Lock expires; message redelivers.

**Sessions** group related messages by `SessionId`. Critical when ordering matters within a customer or per saga instance. Sessions also support **session state** — a key-value blob stored next to the session, perfect for stateful workflows.

---

## Diagram

```mermaid
sequenceDiagram
    participant Producer
    participant Queue as Queue / Topic
    participant Consumer
    participant DLQ as Dead-letter Queue
    Producer->>Queue: SendMessage(body, sessionId?, ttl, props)
    Queue->>Consumer: Receive (peek-lock)
    Consumer->>Consumer: process
    alt Success
        Consumer->>Queue: Complete
    else Transient failure
        Consumer->>Queue: Abandon (redeliver)
    else Poison
        Consumer->>Queue: DeadLetter(reason)
        Queue->>DLQ: move message
    else Lock expires
        Queue->>Queue: lock released; another consumer receives
    end
```

---

## Syntax & API

### Provision a namespace, queue, topic + subscription

```bash
RG=rg-sb-demo
LOC=eastus
NS=sb-orders-$RANDOM

az group create -n $RG -l $LOC
az servicebus namespace create -g $RG -n $NS -l $LOC --sku Standard

# Queue
az servicebus queue create -g $RG --namespace-name $NS -n orders \
  --max-delivery-count 10 \
  --default-message-time-to-live "P14D" \
  --enable-dead-lettering-on-message-expiration true \
  --requires-session false \
  --duplicate-detection-history-time-window "PT10M"

# Topic + filtered subscription
az servicebus topic create -g $RG --namespace-name $NS -n order-events
az servicebus topic subscription create -g $RG --namespace-name $NS \
  --topic-name order-events -n billing
az servicebus topic subscription rule create -g $RG --namespace-name $NS \
  --topic-name order-events --subscription-name billing \
  --name only-paid --filter-sql-expression "EventType = 'OrderPaid'"
```

### .NET — publish and consume

```csharp
// Producer
await using var client = new ServiceBusClient(
    $"{ns}.servicebus.windows.net", new DefaultAzureCredential());
var sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage(JsonSerializer.SerializeToUtf8Bytes(order))
{
    MessageId = order.Id,                   // duplicate detection
    SessionId = order.TenantId,             // ordering per tenant
    TimeToLive = TimeSpan.FromHours(2)
});

// Consumer (sessionful)
var processor = client.CreateSessionProcessor("orders",
    new ServiceBusSessionProcessorOptions { AutoCompleteMessages = false, MaxConcurrentSessions = 4 });

processor.ProcessMessageAsync += async args =>
{
    try
    {
        var order = JsonSerializer.Deserialize<Order>(args.Message.Body);
        await handler.HandleAsync(order!, args.CancellationToken);
        await args.CompleteMessageAsync(args.Message);
    }
    catch (PoisonException ex)
    {
        await args.DeadLetterMessageAsync(args.Message, "Poison", ex.Message);
    }
};
processor.ProcessErrorAsync += a => { log.Error(a.Exception, "SB error"); return Task.CompletedTask; };
await processor.StartProcessingAsync();
```

### Scheduled delivery

```csharp
await sender.ScheduleMessageAsync(
    new ServiceBusMessage("retry-payload"),
    DateTimeOffset.UtcNow.AddMinutes(30));
```

---

## Common Patterns

- **Outbox → Service Bus → Consumer**: app commits with an outbox row; a dispatcher publishes; consumers handle idempotently keyed by `MessageId`.
- **Topic fan-out** for domain events: `OrderPlaced` to multiple subscribers each with filter rules.
- **Saga with sessions**: each saga instance gets a `SessionId`; messages for the saga arrive in order to the same consumer with session state.
- **Retry with delay**: catch transient errors, `Abandon` for immediate retry up to `MaxDeliveryCount`, or `Schedule` a delayed retry copy and `Complete` the original.
- **DLQ monitor**: alert when DLQ depth > 0; investigate poison messages, fix bug, replay.

---

## Gotchas & Tips

- **At-least-once means duplicates happen.** Always design consumers to be idempotent (use `MessageId` as a dedup key in the DB).
- **Lock durations are short (default 30s).** Long-running handlers must call `RenewMessageLockAsync` periodically.
- **Sessions serialize within session.** Bigger fan-out across many small sessions, not one fat session.
- **DLQ messages still cost storage** but don't expire by default. Set a retention policy and a clean-up job.
- **Duplicate detection window is per queue/topic.** If two sends within the window share `MessageId`, the second is silently dropped.
- **Premium uses messaging units (MU)**, not transactions. Predictable performance but higher base cost. Standard is auto-scaled by usage.
- **Max message size**: 256 KB (Standard), 100 MB (Premium with `LargeMessage` claim-check pattern recommended anyway).
- **Topic filters use a SQL-like syntax** evaluated server-side. Avoid heavy filtering on the consumer side.
- **AMQP over WebSockets** is supported for restrictive networks (`TransportType = AmqpWebSockets`).
- **Don't use Service Bus as a database.** It's a broker; long retention + heavy querying is the wrong tool.

---

## See Also

- [[02 - Azure Functions]]
- [[12 - Event Grid]]
- [[13 - Event Hubs]]
- [[14 - Storage Queues vs Service Bus]]
- [[17 - Event-Driven Architecture]]
