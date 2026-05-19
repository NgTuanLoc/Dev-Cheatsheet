---
tags: [business, beginner, logistics, warehouse]
aliases: [WMS, Picking]
level: Beginner
---

# Warehouse Management Pick Pack Ship

> **One-liner**: Inside a warehouse, an order becomes a route — a sequence of bins to visit, items to scan, and a box to pack — and every step is a barcode confirmation.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| WMS | Warehouse Management System |
| Bin / Location | A specific shelf slot (e.g., `A-12-03`) |
| Zone | Cluster of bins assigned to one picker |
| Wave | A batch of orders released together for picking |
| Pick path | The optimized walking route through bins |
| Pick list | Line items grouped by zone, sorted by pick path |
| Scan-to-confirm | Barcode scan at each step (bin + item) |
| Pack | Assemble the picked items into a shippable carton |
| Packing slip | Printed list inside the carton |
| Dispatch / Ship-out | Hand to carrier; manifest closes |
| Cycle count | Periodic small recount (vs full annual inventory) |
| GS1-128 | Standard barcode label for logistics units (SSCC inside) |
| SSCC | Serial Shipping Container Code — 18-digit |
| EDI 940 | Warehouse shipping order (from brand → 3PL) |
| EDI 945 | Warehouse shipping advice (from 3PL → brand) |

---

## Core Concept

Inside a warehouse, an order is not a single "fulfil" command — it is a sequence of physical steps: release a wave of orders together, assign pickers to zones, walk an optimized pick path through bins, scan each bin and item as it goes into the tote, pack the tote into a carton, weigh and label the carton, and drop it on a dispatch lane for the carrier. Every step is barcode-confirmed because the dominant source of inventory inaccuracy is human error — a wrong item picked, a wrong quantity counted, a tote sent to the wrong lane. Scanning forces the system to verify reality at each handoff.

Two metrics dominate warehouse operations. **Lines-per-hour** measures picker productivity and drives staffing, wave size, and pick-path design. **Inventory accuracy** measures whether the WMS count agrees with what is actually on the shelf, and is maintained by cycle counting — small recounts of a few bins per day — rather than by shutting the building down for an annual full inventory. Both metrics live on the warehouse manager's dashboard, and changes to the WMS that regress either of them will be felt within a shift.

When a brand outsources fulfilment to a third-party logistics provider (3PL), integration is typically file-based EDI: 940 outbound (the brand sends the order to the 3PL) and 945 inbound (the 3PL confirms what shipped). These exchanges are asynchronous, often delivered over SFTP, and require careful sequencing.

---

## Syntax & API

```csharp
public sealed record PickListLine(string Sku, string Bin, int Quantity);

public sealed record PickList(
    string Id,
    string PickerId,
    DateOnly WaveDate,
    IReadOnlyList<PickListLine> Lines  // pre-sorted by pick path
);
```

---

## Common Patterns

```csharp
public async Task ConfirmScanAsync(string pickListId, string bin, string sku, int qty, CancellationToken ct)
{
    var line = await _picks.GetExpectedAsync(pickListId, bin, sku, ct)
        ?? throw new InvalidOperationException("Unexpected scan");
    if (qty != line.Quantity) throw new InvalidOperationException("Quantity mismatch");
    await _picks.MarkConfirmedAsync(line.Id, ct);
}
```

---

## Gotchas & Tips

- Don't decrement stock at pick time — decrement at ship time (carton-closed event). A pick can fail.
- Short picks (item not in bin) are common; the system must allow a "pick exception" path that re-allocates or backorders.
- Pickers don't think in SQL — UIs are barcode-scanner-driven, terminal-style.
- Multi-warehouse fulfilment: an order can split — see [[01 - Inventory and Stock Reservations]].

---

## See Also

- [[07 - Shipping and Delivery Basics]]
- [[01 - Inventory and Stock Reservations]]
- [[04 - Order Management Basics]]
