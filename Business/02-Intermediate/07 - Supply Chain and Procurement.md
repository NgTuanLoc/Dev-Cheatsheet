---
tags: [business, intermediate, logistics, supplychain]
aliases: [Procurement, Supply Chain]
level: Intermediate
---

# Supply Chain and Procurement

> **One-liner**: Procurement is "what do we buy, from whom, when, how much" — and the EOQ formula has been the answer since 1913, with caveats.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| Supplier / Vendor | Counterparty selling you goods/services |
| Purchase order (PO) | Authoritative buy commitment — number, lines, terms |
| 3-way match | PO + goods receipt + supplier invoice must agree before pay |
| Lead time | Days from order to receipt |
| MOQ | Minimum Order Quantity from the supplier |
| EOQ | Economic Order Quantity — minimizes ordering + holding cost |
| Safety stock | Buffer for lead-time variability |
| Reorder point | Trigger to place a new PO |
| ABC analysis | Categorize SKUs by revenue contribution (A=top, C=bottom) |
| JIT | Just-In-Time — receive close to use, low inventory |
| Forecast accuracy | MAPE, WMAPE — typical metric for demand planning |
| Bullwhip effect | Demand variance amplifies upstream the supply chain |
| EDI 850 | Purchase Order |
| EDI 855 | PO Acknowledgement |
| EDI 856 | Advance Ship Notice (ASN) |
| Standard formula | EOQ = √(2DS/H), D=demand, S=order cost, H=holding cost |

---

## Core Concept

The procurement cycle is a chain of hand-offs: forecast → reorder point trigger → PO created → supplier acknowledges (EDI 855) → ships (EDI 856 ASN) → goods received → invoice received → 3-way match → pay. Every step is a hand-off between systems or parties, and every hand-off can fail — most procurement bugs are not about the math but about reconciliation across the boundary.

EOQ math gives you the order size that minimizes the sum of ordering cost (placing a PO has a fixed cost) and holding cost (carrying inventory ties up capital). The formula is `EOQ = √(2DS/H)`, derived in 1913 and still the textbook starting point. In practice MOQs from suppliers and pricing tiers (buy 1000 for 10% off) override the math, but it's still a useful anchor when negotiating with finance.

Bullwhip is the classic supply-chain pathology: end-customer demand fluctuates slightly, but each tier upstream over-reacts, so the manufacturer at the back of the chain sees huge swings on top of tiny real changes. Information sharing (POS data flowing back to the supplier) and smaller, more frequent orders dampen it. Big batch ordering amplifies it.

---

## Syntax & API

```csharp
public sealed record PurchaseOrder(
    string Id,
    string SupplierId,
    DateOnly OrderDate,
    DateOnly ExpectedDeliveryDate,
    IReadOnlyList<POLine> Lines,
    Money Total
);

public sealed record POLine(string Sku, int Quantity, Money UnitCost);
```

---

## Common Patterns

```csharp
public static int EconomicOrderQuantity(int annualDemand, decimal orderCost, decimal holdingCostPerUnitPerYear)
    => (int)Math.Ceiling(Math.Sqrt((double)(2m * annualDemand * orderCost / holdingCostPerUnitPerYear)));

public static int ReorderPoint(int dailyDemand, int leadTimeDays, int safetyStock)
    => dailyDemand * leadTimeDays + safetyStock;
```

---

## Gotchas & Tips

- 3-way match mismatches are normal — define a tolerance (e.g., ±2% on amount, ±5% on qty) and an exception queue.
- Lead-time variability matters more than mean lead time — high variance drives bigger safety stock.
- Forecast bias (consistent over- or under-prediction) is more harmful than noise — track and correct.
- Single-source vs multi-source is a resilience choice — single is cheaper, multi survives disruption.

---

## See Also

- [[01 - Inventory and Stock Reservations]]
- [[08 - Warehouse Management Pick Pack Ship]]
- [[07 - Manufacturing BOM and MRP]]
