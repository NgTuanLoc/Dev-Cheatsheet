---
tags: [business, intermediate, commerce, pricing]
aliases: [Pricing, Promotions]
level: Intermediate
---

# Pricing Discounts and Promotions

> **One-liner**: Price isn't one number — it's a stack of rules applied in a specific order, and "stacking" is the bug factory.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| List price | The default catalog price |
| Sale price | A scheduled override on list price |
| Price list | Per-segment overrides (B2B tier, region, currency) |
| Coupon / Promo code | Customer-entered token that applies a rule |
| Stacking | Whether multiple promotions can apply to one order |
| Tier discount | Buy N, get N% off (linear or step) |
| BOGO | Buy-one-get-one (free or N% off) |
| Threshold | Free shipping if subtotal > X |
| Dynamic pricing | Algorithmic price changes (demand, time, competitor) |
| Markdown | Permanent price drop on slow-moving stock |
| Order of operations | line discount → cart discount → shipping discount → tax |
| Rule engine | DSL or library (e.g., Sandstorm Rules, RulesEngine) |
| Attribution | Which promotion gets credited for the order? |

---

## Core Concept

Pricing logic compounds quickly. The simplest version is a static catalog price. The next layer is scheduled sales. Then customer-specific price lists (loyalty tier, employee, wholesale). Then coupons, which can be percent-off, flat-amount off, BOGO, free-shipping, free-gift, or combinations of the above. The combinatorial explosion is real, and a small ruleset on paper becomes a large test matrix in code.

The single biggest correctness rule is **the order of application**. Line discounts apply first (the customer sees a discounted line), then cart-level discounts (e.g., "10% off orders over $50"), then shipping, then tax. Applying these in the wrong order changes the final order total, and silently — the bug only shows up when finance reconciles.

Stacking rules — which promotions can combine — are a business decision, not a technical one. Push for explicit rules ("at most one storewide promo + one shipping promo + unlimited line promos") rather than ad-hoc behavior. Once the rules are explicit, the engine can be a small piece of code instead of a sprawling pile of conditionals.

---

## Syntax & API

```csharp
public abstract record Discount(string Code);
public sealed record PercentOffLine(string Code, string Sku, decimal Percent) : Discount(Code);
public sealed record FlatOffCart(string Code, Money Amount) : Discount(Code);
public sealed record FreeShipping(string Code, decimal MinSubtotal) : Discount(Code);
```

---

## Common Patterns

```csharp
public OrderTotals Calculate(Cart cart, IReadOnlyList<Discount> discounts)
{
    var lines = cart.Lines.Select(l => ApplyLineDiscounts(l, discounts)).ToList();
    var subtotal = lines.Sum(l => l.LineTotal.Amount);
    var cartDiscount = ApplyCartDiscounts(subtotal, discounts);
    var shipping = ApplyShippingDiscounts(cart, discounts);
    var taxable = subtotal - cartDiscount + shipping;
    var tax = _tax.For(taxable, cart.ShipTo);
    return new OrderTotals(subtotal, cartDiscount, shipping, tax, taxable + tax);
}
```

---

## Gotchas & Tips

- Tax is computed on the discounted amount in most jurisdictions, but **not all** — check by jurisdiction.
- Coupon codes are case-insensitive in user expectation; store canonicalized.
- Per-customer usage limits require a `(coupon_code, customer_id)` constraint, not just a global counter.
- Promotion attribution affects marketing reporting — agree on the rule (last-touch, first-touch, weighted) before launching.

---

## See Also

- [[02 - Ecommerce Storefront Foundations]]
- [[03 - Subscriptions and Recurring Billing]]
- [[02 - Loyalty and Rewards Programs]]
