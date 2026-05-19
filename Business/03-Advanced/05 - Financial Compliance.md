---
tags: [business, advanced, finance, compliance]
aliases: [Compliance, PSD2, MiFID]
level: Advanced
---

# Financial Compliance

> **One-liner**: Financial compliance is "prove to a regulator you ran the right control at the right time" — and proving means audit trails, immutable logs, and reports nobody reads until they do.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| PSD2 | EU Payment Services Directive 2 — SCA, open banking APIs |
| SCA | Strong Customer Authentication — 2 of (knowledge, possession, inherence) |
| Open Banking | Account info + payment initiation APIs (PSD2 art. 66/67) |
| MiFID II | EU markets — best execution, transaction reporting (RTS 22), unbundling |
| Best execution | Obligation to take all sufficient steps for best client result |
| Basel III | Bank capital + liquidity — LCR, NSFR, leverage ratio |
| Dodd-Frank | US — swap reporting, Volcker rule, CFTC/SEC powers |
| GDPR | EU data protection — applies to financial PII |
| MAR | Market Abuse Regulation — insider dealing, market manipulation |
| Transaction reporting | RTS 22 — submit fields per executed trade |
| Audit trail | Append-only log of every state-changing action |
| Record retention | 5–7 years typical — varies by regulation |
| Suspicious activity | Filing duty (SAR) — see [[04 - KYC AML and Sanctions Screening]] |
| 4th-line defence | Internal audit — independent verification of controls |

---

## Core Concept

Compliance is a product feature, not an afterthought bolted on at the end. Every regulation maps to specific engineering surfaces. PSD2's Strong Customer Authentication lives in the checkout flow and step-up authentication. MiFID II best execution lives in the venue-selection logic and the RTS 27/28 reports that justify those routing decisions. Basel III capital requirements live in the calculation engines and daily reports that prove the bank stays above its thresholds. Treating these as features means they show up in the architecture diagram and the sprint backlog, not in the compliance department's spreadsheets.

The two universal engineering primitives across all of these regulations are the **audit trail** and **immutability**. An audit trail is an append-only event log capturing who did what, when, and why for every state change that matters. Immutability means records cannot be edited in place — corrections are made by reversing entries that themselves are logged. These primitives are how you survive a regulator's inquiry months or years after the fact.

Regulatory reporting is a discipline of its own. Fields are tightly specified, timing is strict (RTS 22 is T+1 by 23:59 local time), and late filing draws fines. Build the reporting pipeline as a first-class deliverable from the start, with the same engineering rigour as customer-facing systems — not as a quarterly script someone runs from a laptop.

---

## Syntax & API

```csharp
public sealed record AuditEvent(
    string Id,
    DateTimeOffset At,
    string Actor,         // user or system id
    string Action,        // "ORDER_PLACED", "MARGIN_CALL_ISSUED"
    string EntityType,
    string EntityId,
    string Before,        // JSON
    string After,         // JSON
    IReadOnlyDictionary<string, string> Metadata
);
```

---

## Common Patterns

```csharp
public sealed record Rts22Report(
    string TransactionReferenceNumber, // field 1
    DateTimeOffset TradingDateTime,    // field 28
    string Isin,                       // field 41
    decimal Quantity,                  // field 30
    decimal Price,                     // field 33
    string BuyerId,                    // fields 7+8
    string SellerId,                   // fields 16+17
    string ExecutionVenue              // field 36 — MIC code
    // ... 65 fields total ...
);
```

---

## Gotchas & Tips

- Audit logs are **not** application logs. Application logs can be sampled, summarized, and deleted; audit logs cannot.
- Immutability is enforced at storage (append-only tables, WORM storage) and at code level (no UPDATE statements on regulatory tables).
- Regulator timing is unforgiving — RTS 22 reports are due T+1 by 23:59 local time; PSD2 SCA failures must be retried correctly within a session.
- Cross-regulator overlap: a single trade can hit MiFID, EMIR, and MAR — you may report it multiple times with overlapping but different fields.

---

## See Also

- [[04 - KYC AML and Sanctions Screening]]
- [[03 - FX Trading and Settlement]]
- [[06 - Payments Card Processing and Gateways]]
