---
tags: [business, beginner, commerce]
aliases: [Storefront, Ecommerce Basics]
level: Beginner
---

# Ecommerce Storefront Foundations

> **One-liner**: Showing products, putting them in a cart, and collecting payment — the three jobs every storefront has, and the data each one mutates.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| Catalog | Browsable list of products (read-mostly) |
| Cart | Mutable list of (SKU, qty, price snapshot) per session |
| Checkout | Multi-step: address → shipping option → payment → review → place |
| Session | Cart key — anonymous cookie or authenticated user id |
| Guest checkout | Place order without account; common for first-time buyers |
| Abandoned cart | Cart with no order placed within N hours |
| Conversion rate | Orders placed / sessions started |
| AOV | Average Order Value = revenue / orders |
| Price snapshot | Cart stores price-at-add-time, not catalog price (catalog can change) |
| Inventory check | At checkout, not at cart-add (UX cost vs oversell cost) |
| Standard APIs | Shopify Storefront, BigCommerce, Magento REST |
| Webhook events | `cart.created`, `cart.updated`, `checkout.completed`, `order.created` |

---

## Core Concept

The storefront is a thin facade over three domain services: the catalog, the cart, and the order pipeline. The most common mistake new engineers make is putting business logic into the storefront itself — pricing rules, inventory validation, tax calculation — when those concerns belong in domain services that the storefront simply calls. Keep the storefront stateless about what *should* happen; let it reflect what the domain says.

Price snapshotting is the correctness rule most likely to be missed. The cart must store the exact price the customer saw at the moment they added the item, not the current catalog price. Catalog prices change all the time (promotions start, vendor cost updates land, currency rates shift); pulling the live price at checkout silently changes the deal the customer agreed to. The snapshot is the contract.

Three numbers dominate every storefront review: conversion rate (orders divided by sessions), average order value (revenue divided by orders), and abandoned-cart rate. Product and business stakeholders watch these in real time. Any change you ship — a redesigned button, a new checkout step, a different inventory rule — that regresses any of these will surface quickly, so think about measurement before you change UX.

---

## Syntax & API

```csharp
public sealed record CartLine(string Sku, int Quantity, Money PriceAtAdd);

public sealed class Cart
{
    public string SessionId { get; init; } = default!;
    public List<CartLine> Lines { get; } = new();

    public Money Subtotal => Lines
        .Select(l => new Money(l.PriceAtAdd.Amount * l.Quantity, l.PriceAtAdd.Currency))
        .Aggregate((a, b) => a + b);
}
```

---

## Common Patterns

```csharp
public sealed class AbandonedCartJob
{
    public async Task RunAsync(CancellationToken ct)
    {
        var cutoff = DateTimeOffset.UtcNow.AddHours(-24);
        var stale = await _carts.FindAbandonedSince(cutoff, ct);
        foreach (var cart in stale)
            await _email.SendReminderAsync(cart.SessionId, ct);
    }
}
```

---

## Gotchas & Tips

- Never re-query catalog price at order-place time — use the snapshot.
- Cart total includes neither tax nor shipping; recompute at checkout.
- Session expiry should be longer than typical "think time" — 30 days is common for B2C.
- Webhooks are at-least-once; consumers must be idempotent on `event.id`.

---

## See Also

- [[03 - Product Catalog and SKU Model]]
- [[04 - Order Management Basics]]
- [[02 - Pricing Discounts and Promotions]]
