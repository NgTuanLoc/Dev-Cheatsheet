---
tags: [business, beginner]
aliases: [Business Overview, Domain Overview]
level: Beginner
---

# Business Domain Overview

> **One-liner**: A working engineer hits commerce, finance, and logistics concerns on almost every job — this topic gives you the vocabulary, the standards, and the failure modes for each.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| Three families | Financial Services, Commerce & Retail, Logistics & Operations |
| Default code language | `csharp` |
| Money type | `decimal` + `Currency` (ISO 4217, e.g. `USD`, `EUR`) |
| Business date type | `DateOnly` |
| Event time type | `DateTimeOffset` (always UTC at rest) |
| Ledger rule | Sum of debits == sum of credits in every transaction |
| State-machine rule | Domain entities have **explicit** statuses; transitions are validated |
| Idempotency key location | Header `Idempotency-Key` on every state-changing API call |
| Outbox pattern | Persist event in same DB transaction as state change; relay later |
| Read this before | Touching any code involving money, orders, shipments, or policies |

---

## Core Concept

Almost every product eventually grows a commerce surface (catalog, cart, checkout), a finance surface (payments, ledger, reporting), or a logistics surface (shipments, inventory, fulfilment). Each of these has decades of accumulated industry vocabulary and standards behind it. Knowing the existing terminology saves you from reinventing names — and, more importantly, from reinventing semantics that have already been argued about and codified.

The three families are wider than they look, but they share a small set of recurring patterns. Money is `decimal` paired with a currency, never `double`. State changes go through explicit transitions on aggregates, not through raw setters. Integrations are idempotent, keyed off a header or an event id, because every real network is at-least-once. Events that mutate state are persisted in the same transaction as the change (the outbox pattern) so the world cannot disagree with the database.

These notes are scoped to what an engineer needs at the boundary between code and the domain. They will not replace an actuarial, accounting, or customs textbook — but they will tell you what to search for, which standards apply, and which inventions to avoid. When in doubt, the rule is "look for the ISO/PCI/GS1 standard before writing your own."

---

## Syntax & API

```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return new Money(a.Amount + b.Amount, a.Currency);
    }
}
```

---

## Common Patterns

```csharp
public enum OrderStatus { Placed, Paid, Shipped, Delivered, Cancelled }

public sealed class Order
{
    public OrderStatus Status { get; private set; }

    public void MarkPaid()
    {
        if (Status != OrderStatus.Placed)
            throw new InvalidOperationException($"Cannot pay an order in status {Status}");
        Status = OrderStatus.Paid;
    }
}
```

---

## Gotchas & Tips

- Never store money in `double` / `float`.
- Currency is always part of a `Money` value — never just an amount on its own.
- State transitions go through methods, not setters.
- Timestamps in UTC at rest; convert at the UI boundary only.

---

## See Also

- [[Business Master Index]]
- [[06 - Payments Card Processing and Gateways]]
- [[06 - Accounting Ledger and Double-Entry]]
