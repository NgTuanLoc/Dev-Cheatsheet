---
tags: [business, advanced, commerce, loyalty]
aliases: [Loyalty, Points]
level: Advanced
---

# Loyalty and Rewards Programs

> **One-liner**: A loyalty programme is a parallel currency you issue — and like any currency, you have to balance issuance, redemption, expiration, and the float between them.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| Points | The loyalty currency (proprietary, non-redeemable for cash) |
| Earn rate | Points per dollar spent (or per action) |
| Burn / Redemption | Spending points for rewards |
| Conversion rate | Points-to-dollars value (issuer-controlled) |
| Tier | Status level (Bronze/Silver/Gold) tied to spend or visits |
| Status period | Window over which spend is measured for tier (often rolling 12m) |
| Earn velocity | Points earned per period — tier-driven |
| Breakage | Points issued but never redeemed |
| Liability | Points outstanding × accrual value (financial liability on balance sheet) |
| Expiration | Points expire after N months of inactivity (or absolutely) |
| Partner earn | Earn points outside your direct sale (co-branded) |
| Float | Points issued − points redeemed at a given time |
| Standard accounting | Defer revenue per IFRS 15 / ASC 606 attributable to points |

---

## Core Concept

Loyalty is finance pretending to be marketing. Every point you issue is a liability on the balance sheet — you owe the customer something of value — and modern accounting rules (IFRS 15 and ASC 606) require deferring revenue corresponding to the points portion of each sale. Sell $100 with 100 points worth $5, and only $95 is revenue today; the $5 is deferred until the points are burned or expire.

Tiers are the most engagement-effective lever in the toolkit. Customers will overspend to keep a status they have already earned, and they will track progress toward the next tier obsessively. The status-period rule (rolling 12 months vs calendar year vs lifetime) materially changes that behaviour and has to be a deliberate product decision, not an implementation accident.

Breakage is the platform's bet. Not every point gets redeemed — some are forgotten, some expire, some belong to lapsed customers. Estimating breakage rates is actuarial work: typically 10–25% in retail, lower in airlines where points are more valuable per unit. The estimate feeds directly into the deferred-revenue calculation, so mis-estimating breakage creates accounting risk and restatements. Float — points issued minus points redeemed — is the operational mirror of the liability.

---

## Syntax & API

```csharp
public enum PointMovement { Earn, Burn, Adjustment, Expire }

public sealed record PointEntry(
    string Id,
    string CustomerId,
    PointMovement Movement,
    int Amount,           // positive = earn, negative = burn
    DateTimeOffset At,
    string RefId          // order id, redemption id, etc.
);

public static int Balance(IEnumerable<PointEntry> entries, DateTimeOffset asOf)
    => entries.Where(e => e.At <= asOf).Sum(e => e.Amount);
```

---

## Common Patterns

```csharp
public static string TierForSpend(decimal spendL12m) => spendL12m switch
{
    >= 5000m => "Gold",
    >= 1500m => "Silver",
    _        => "Bronze"
};
```

---

## Gotchas & Tips

- Points are not cash equivalents — never let your UI imply they are; tax authorities and regulators care.
- Expiration must be clearly communicated and uniformly enforced; ambiguity invites consumer protection complaints.
- IFRS 15: when you sell $100 with 100 points worth $5, recognize $95 revenue + $5 deferred until points redeem or expire.
- Co-branded earn introduces a B2B settlement leg (partner pays you for issued points).

---

## See Also

- [[02 - Pricing Discounts and Promotions]]
- [[06 - Accounting Ledger and Double-Entry]]
- [[03 - Subscriptions and Recurring Billing]]
