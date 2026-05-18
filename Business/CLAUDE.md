# CLAUDE.md — Business Topic

Topic-specific guidance for the `Business/` section of the vault. For shared rules (template, Mermaid, authoring conventions), see the root [CLAUDE.md](../CLAUDE.md).

## What This Topic Is

A developer-facing cheatsheet for the business domains a working engineer will hit on the job — **Financial Services**, **Commerce & Retail**, and **Logistics & Operations**. Notes explain the domain terms, the standard APIs, the regulatory standards, and the parts where engineers get it wrong. Code examples are C# / .NET because the vault is .NET-focused, but the domain concepts are language-agnostic.

---

## Folder Structure

```
Business/
├── CLAUDE.md                    ← this file
├── 00-Index/
│   ├── Business Master Index.md ← single entry point
│   └── Business Learning Path.md ← recommended reading order
├── 01-Beginner/                 (8 notes — domain primers)
├── 02-Intermediate/             (8 notes — workflows, integrations, money math, logistics ops)
└── 03-Advanced/                 (8 notes — regulated and distributed-system concerns)
```

---

## Tag Taxonomy (Business-specific)

Always include `business` plus the level. Add one **family** tag and one **subdomain** tag.

| Tag | Meaning |
|-----|---------|
| `business` | applied to every Business note |
| `finance` | family tag — Financial Services notes |
| `commerce` | family tag — Commerce & Retail notes |
| `logistics` | family tag — Logistics & Operations notes |
| `payments` | subdomain — card / wallet / gateway processing |
| `banking` | subdomain — accounts, transfers, statements |
| `fx` | subdomain — FX, trading, settlement |
| `lending` | subdomain — credit, loans, scoring |
| `insurance` | subdomain — policies, claims, reserves |
| `accounting` | subdomain — ledger, double-entry, COA |
| `compliance` | subdomain — KYC/AML, PSD2, MiFID, Basel |
| `inventory` | subdomain — stock, ATP, oversell |
| `pricing` | subdomain — discounts, promotions, dynamic pricing |
| `subscriptions` | subdomain — recurring billing, dunning, churn |
| `marketplace` | subdomain — sellers, payouts, take rate |
| `loyalty` | subdomain — points, tiers, redemption |
| `shipping` | subdomain — carriers, labels, tracking |
| `warehouse` | subdomain — WMS, pick/pack/ship |
| `supplychain` | subdomain — procurement, suppliers, forecasting |
| `fleet` | subdomain — routing, last-mile, telematics |
| `manufacturing` | subdomain — BOM, MRP, MES |
| `customs` | subdomain — cross-border, Incoterms, HS codes |

---

## Code-Block Conventions

| Lang tag | Use for |
|----------|---------|
| `csharp` | domain models, services, integration handlers |
| `sql` | schema sketches, sample queries |
| `bash` | CLI / curl examples against payment / shipping APIs |
| `json` | API request/response payloads, webhook bodies |
| `mermaid` | diagrams |

Examples must be **minimal and compilable**. Money is always `decimal` with a `Currency` companion field. Dates are `DateOnly` for business dates and `DateTimeOffset` for event times.

---

## Topic Coverage Map

### Beginner (01-Beginner) — 8 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Business Domain Overview | Why this topic; how to use these notes; family tags |
| 02 | Ecommerce Storefront Foundations | Catalog, cart, checkout, session, conversion |
| 03 | Product Catalog and SKU Model | SKU, variants, GTIN, taxonomy, attributes |
| 04 | Order Management Basics | Order lifecycle, statuses, line items, fulfillment handoff |
| 05 | Retail Banking Accounts and Transfers | Accounts, IBAN/sort code, balance, posting, statements |
| 06 | Payments Card Processing and Gateways | Auth/capture/refund, 3DS, tokenization, PCI DSS |
| 07 | Shipping and Delivery Basics | Carriers, labels, tracking, rate shopping |
| 08 | Warehouse Management Pick Pack Ship | Bins, waves, picking, packing, dispatch |

### Intermediate (02-Intermediate) — 8 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Inventory and Stock Reservations | On-hand vs reserved vs ATP, oversell, multi-warehouse |
| 02 | Pricing Discounts and Promotions | Price lists, coupons, stacking, dynamic pricing |
| 03 | Subscriptions and Recurring Billing | Plans, proration, dunning, MRR/ARR, churn |
| 04 | KYC AML and Sanctions Screening | Identity, PEP, OFAC, ongoing monitoring, SAR |
| 05 | Lending and Credit Scoring | Origination, scorecard, amortization, NPL, IFRS 9 ECL |
| 06 | Accounting Ledger and Double-Entry | Journals, debits/credits, COA, reconciliation |
| 07 | Supply Chain and Procurement | Suppliers, POs, lead time, MOQ, EOQ, forecasting |
| 08 | Fleet and Last-Mile Delivery | Routing, VRP, telematics, proof-of-delivery |

### Advanced (03-Advanced) — 8 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Marketplaces and Multi-Sided Platforms | Sellers, KYB, escrow, payouts, take rate, disputes |
| 02 | Loyalty and Rewards Programs | Points, accrual, tiers, redemption, breakage |
| 03 | FX Trading and Settlement | Order book, spread, T+2, CLS, FIX protocol |
| 04 | Derivatives Risk and Margin | Futures, options, Greeks, VaR, margin call |
| 05 | Financial Compliance | PSD2 SCA, MiFID II, Basel III, audit trail |
| 06 | Insurance Policy Underwriting and Claims | Policy lifecycle, underwriting, claims, reserves |
| 07 | Manufacturing BOM and MRP | BOM, work orders, MRP, MES, OEE |
| 08 | Cross-Border Logistics and Customs | Incoterms, HS codes, duties, customs declarations |

**Total: 24 notes** (8 beginner + 8 intermediate + 8 advanced)

---

## Diagram Coverage Targets

These notes specifically benefit from diagrams and **must** include one:

| Note | Diagram type |
|------|--------------|
| Order Management Basics | stateDiagram-v2 (order status transitions) |
| Payments Card Processing and Gateways | sequenceDiagram (auth → capture → settlement) |
| Inventory and Stock Reservations | flowchart (on-hand vs reserved vs ATP math) |
| Subscriptions and Recurring Billing | stateDiagram-v2 (subscription lifecycle + dunning) |
| KYC AML and Sanctions Screening | flowchart (onboarding decision tree) |
| Accounting Ledger and Double-Entry | classDiagram (Journal → Entry → Account) |
| FX Trading and Settlement | sequenceDiagram (trade lifecycle T+0 → T+2) |
| Derivatives Risk and Margin | flowchart (margin call decision) |
| Insurance Policy Underwriting and Claims | stateDiagram-v2 (policy + claim states) |
| Manufacturing BOM and MRP | erDiagram (Product → BOM → Component) |
| Cross-Border Logistics and Customs | sequenceDiagram (shipper → carrier → customs → consignee) |
| Marketplaces and Multi-Sided Platforms | graph (buyer ↔ platform ↔ seller money flow) |

---

## Cross-Topic Links

Business notes routinely cross-link into other vault topics:

- Money modelling, ledger correctness, idempotent handlers → `Dotnet` topic
- Schema design for orders / accounts / inventory, transactional outbox → `Database` topic
- Hosting & secrets for payment integrations → `Azure` topic

Always include at least one such cross-link in `## See Also` when applicable.