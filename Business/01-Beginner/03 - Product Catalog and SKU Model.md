---
tags: [business, beginner, commerce]
aliases: [Catalog, SKU]
level: Beginner
---

# Product Catalog and SKU Model

> **One-liner**: A product is what marketing puts on a page; a SKU is what a warehouse picks. Get the distinction right and the rest of commerce slots in.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| Product | Marketing object — title, description, images |
| SKU | Stock Keeping Unit — the *thing you actually sell and ship* |
| Variant | A SKU within a product family (e.g., size M, color red) |
| Option | The axis of variance (size, color) |
| Attribute | A facet for filtering/search (brand, material, GTIN) |
| GTIN | Global Trade Item Number — the barcode you scan |
| UPC | 12-digit GTIN — North America |
| EAN | 13-digit GTIN — Europe |
| MPN | Manufacturer Part Number — vendor's id, not unique cross-vendor |
| ASIN | Amazon-specific id |
| Taxonomy | Hierarchical category tree (often Google Product Taxonomy) |
| GS1 | Standards body that issues GTIN prefixes |
| Standard schema | schema.org/Product for SEO markup |

---

## Core Concept

A "product" in business terms is the marketing object. It carries the title, the description, the lifestyle photos, and the category — everything the customer sees on a detail page. It is explicitly *not* the thing you ship. What you ship is the **SKU**, the Stock Keeping Unit, which is the smallest sellable unit your business tracks. A single product can have many SKUs: a t-shirt page is one product with SKUs for every size and color combination.

Identifier discipline matters here. GTIN, UPC, and EAN are *external* identifiers shared across the entire supply chain — manufacturers, retailers, marketplaces, and shipping systems all reference them. SKU is *your* internal identifier, owned by you, stable for the life of the SKU. Never confuse the two: GTIN tells the world what the item is; SKU tells your warehouse which slot to pick from.

Attributes do double duty. They drive the customer-facing experience — search facets, filter pills, comparison tables — but they also feed downstream systems. Shipping needs weight and dimensions. Customs needs an HS code. Accounting needs a tax category. Compliance needs hazmat or age-restriction flags. Treat attribute schemas as cross-functional contracts, not just merchandising metadata.

---

## Syntax & API

```csharp
public sealed record Sku(string Code, string ProductId, Money Price, Dictionary<string, string> Options);

public sealed record Product(
    string Id,
    string Title,
    string Description,
    IReadOnlyList<Sku> Skus,
    IReadOnlyDictionary<string, string> Attributes
);
```

---

## Common Patterns

```csharp
public static string GenerateSku(string productCode, IDictionary<string, string> options)
    => $"{productCode}-{string.Join("-", options.OrderBy(o => o.Key).Select(o => o.Value))}";
```

---

## Gotchas & Tips

- SKU code must be stable for the life of the SKU — don't encode mutable attributes (price, supplier).
- A product without variants still has at least one SKU.
- Multi-pack and bundle SKUs need a "components" relationship — not just attributes.
- GTIN is **not** unique within your tenant if you carry the same product from multiple vendors.

---

## See Also

- [[02 - Ecommerce Storefront Foundations]]
- [[04 - Order Management Basics]]
- [[01 - Inventory and Stock Reservations]]
