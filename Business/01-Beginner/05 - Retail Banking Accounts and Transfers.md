---
tags: [business, beginner, finance, banking]
aliases: [Banking Basics, Accounts]
level: Beginner
---

# Retail Banking Accounts and Transfers

> **One-liner**: A bank account is a ledger row with strict rules — every movement is a posting, every posting balances, and "balance" is a computed view, not a stored field you mutate.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| Account | A ledger anchor — has an owner, a currency, a type (current/savings/loan) |
| IBAN | International Bank Account Number — up to 34 chars, country prefix |
| BIC / SWIFT | 8 or 11 char bank identifier |
| Sort code | UK 6-digit bank/branch id |
| ABA / Routing | US 9-digit routing number |
| Balance | Sum of postings up to a cutoff time (never stored as a mutable field) |
| Available balance | Balance − pending holds |
| Posting | One side of a journal entry against an account |
| Value date | The date funds become available (may lag posting date) |
| Statement | Postings within a date range, grouped + opening/closing balance |
| Hold | Reserved funds (e.g., card auth); released or captured later |
| Overdraft | Account allowed to go negative up to a limit |
| Rail (UK) | Faster Payments, BACS, CHAPS, SEPA, SWIFT |
| Standard schema | accounts + postings + holds (3 tables) |
| Standard format | ISO 20022 (pacs.008 for credit transfer) |

---

## Core Concept

A bank account is not a field on a customer record. It is a ledger position — an anchor that owns a stream of immutable postings whose running sum is the balance. This is the fundamental rule that separates a banking system from a CRUD application. Storing balance as a mutable column and updating it on every transaction is how production systems lose money: a crash between debit and credit, or a race between concurrent writers, leaves the books wrong with no audit trail to recover from.

Every operation that moves money is at least two postings, one debit and one credit, linked by a shared transaction id. One-sided postings are illegal in a double-entry system; the invariant "sum of postings for a transaction id equals zero" must hold in every query and every reconciliation. This is not bookkeeping pedantry — it is the only mechanism that lets you detect missing or duplicated entries.

"Available balance" and "balance" are different numbers, and customers care about both. The balance is the sum of all posted entries. The available balance subtracts holds — card authorizations not yet captured, pending outgoing wires, cheque holds while they clear. A customer who sees "balance £500" but a payment declined because available is £200 is a customer who will call support, so surface both numbers in any UI.

---

## Syntax & API

```csharp
public sealed record Posting(
    string Id,
    string AccountId,
    Money Amount,           // positive = credit, negative = debit
    DateTimeOffset PostedAt,
    DateOnly ValueDate,
    string TransactionId    // links the two sides
);

public static Money BalanceAt(IEnumerable<Posting> postings, DateTimeOffset cutoff)
    => postings.Where(p => p.PostedAt <= cutoff)
               .Select(p => p.Amount)
               .Aggregate((a, b) => a + b);
```

---

## Common Patterns

```csharp
public async Task TransferAsync(string fromId, string toId, Money amount, CancellationToken ct)
{
    using var tx = await _db.BeginTransactionAsync(ct);
    var txnId = Guid.NewGuid().ToString();
    await _db.AppendAsync(new Posting(NewId(), fromId, new Money(-amount.Amount, amount.Currency), DateTimeOffset.UtcNow, DateOnly.FromDateTime(DateTime.UtcNow), txnId), ct);
    await _db.AppendAsync(new Posting(NewId(), toId,   amount,                                        DateTimeOffset.UtcNow, DateOnly.FromDateTime(DateTime.UtcNow), txnId), ct);
    await tx.CommitAsync(ct);
}
```

---

## Gotchas & Tips

- Never store balance as a mutable column. Materialize it as a view or a cache that can be rebuilt from postings.
- Two postings per transaction always; check sum-by-`TransactionId` == 0 as an invariant.
- Currency mismatch on a transfer requires an FX leg — see [[03 - FX Trading and Settlement]].
- Settlement on most retail rails is not instant; statement-vs-real-time balance differ.

---

## See Also

- [[06 - Accounting Ledger and Double-Entry]]
- [[06 - Payments Card Processing and Gateways]]
- [[04 - KYC AML and Sanctions Screening]]
