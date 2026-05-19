---
tags: [business, beginner, logistics, shipping]
aliases: [Shipping, Carriers]
level: Beginner
---

# Shipping and Delivery Basics

> **One-liner**: Hand a package to a carrier, give the customer a tracking number, react to status updates — the surface is small; the integrations are where it gets messy.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| Carrier | UPS, FedEx, DHL, USPS, Royal Mail, DPD, etc. |
| Service level | Standard, Express, Overnight, Saturday, Economy |
| AWB | Air Waybill — international shipment id |
| Tracking number | Carrier-issued id on the label |
| Rate shop | Query multiple carriers; pick cheapest meeting SLA |
| Label | PDF/PNG/ZPL the warehouse prints + applies |
| Manifest | End-of-day list handed to the carrier driver |
| Pickup | Carrier collects from warehouse (vs drop-off) |
| Tracking event | Carrier-pushed status (Picked up, In transit, Out for delivery, Delivered) |
| POD | Proof of Delivery — signature/photo/code |
| EDI 204 | Motor carrier load tender |
| EDI 214 | Carrier shipment status |
| Aggregator | One API → many carriers — ShipEngine, EasyPost, Shippo |
| Address validation | USPS / Postcode Anywhere / Google APIs — required before label |

---

## Core Concept

A shipment is born when an order is allocated and picked. From that point the system asks one or more carriers for a rate, picks one based on cost and service-level constraints, requests a label, hands the printed label to the warehouse to apply, and the package goes on a truck. After dispatch the entire surface becomes event-driven: the carrier pushes status updates as the package moves, and your system mirrors them to the customer through tracking pages, notifications, and the order timeline.

Two non-obvious costs catch junior engineers off guard. The first is address validation. Carriers will reject malformed or undeliverable addresses, sometimes after the label has already been issued and you have already been billed for it. Validate the address against an authoritative service (USPS, Postcode Anywhere, Google) before requesting the label, not after. The second is dimensional weight. Carriers bill the *greater* of actual weight and `L×W×H/divisor`, so a light but bulky box can cost more than a heavier compact one. Modelling weight without dimensions guarantees billing surprises.

Rate shopping is a real-time decision but it doesn't have to be a real-time external call for every order. Cache the SLA matrix — which carriers can hit which delivery windows from which origins — and call out only for the live price. This keeps checkout latency predictable.

---

## Syntax & API

```csharp
public sealed record Shipment(
    string Id,
    string OrderId,
    string Carrier,
    string ServiceLevel,
    string TrackingNumber,
    DateOnly ShipDate,
    Address From,
    Address To,
    decimal WeightKg,
    Dimensions Dimensions
);

public sealed record Dimensions(decimal LengthCm, decimal WidthCm, decimal HeightCm);
```

---

## Common Patterns

```bash
curl https://api.easypost.com/v2/shipments \
  -u "$EASYPOST_KEY:" \
  -d "shipment[to_address][zip]=94107" \
  -d "shipment[from_address][zip]=10001" \
  -d "shipment[parcel][weight]=15.4"
```

---

## Gotchas & Tips

- Dimensional weight: `volumetric = L×W×H / divisor` (carriers use 5000 or 6000 cm³/kg). Bill is `max(actual, volumetric)`.
- Once a label is issued you typically have ~28 days before the carrier voids it — track this; voided labels = lost revenue.
- Tracking events are at-least-once and out-of-order — sort by carrier timestamp, dedupe on `(carrier, tracking_number, event_code, timestamp)`.
- International shipments require customs paperwork — see [[08 - Cross-Border Logistics and Customs]].

---

## See Also

- [[08 - Warehouse Management Pick Pack Ship]]
- [[04 - Order Management Basics]]
- [[08 - Fleet and Last-Mile Delivery]]
- [[08 - Cross-Border Logistics and Customs]]
