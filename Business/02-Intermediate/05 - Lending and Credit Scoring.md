---
tags: [business, intermediate, finance, lending]
aliases: [Lending, Credit]
level: Intermediate
---

# Lending and Credit Scoring

> **One-liner**: Lending is "decide who to lend to, decide on what terms, schedule the repayments, recognize the bad ones" — four steps, each its own discipline.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| Origination | The application + decision + funding flow |
| Underwriting | Decision to approve, reject, or counter-offer |
| Scorecard | Statistical model (often logistic regression) producing a score |
| FICO | US consumer credit score, 300–850 |
| VantageScore | Competing US score, also 300–850 |
| DTI | Debt-to-Income ratio — used in mortgage underwriting |
| LTV | Loan-to-Value — loan amount / asset value |
| APR | Annual Percentage Rate — total cost of credit incl. fees |
| Amortization | Schedule of principal + interest payments |
| Schedule types | Equal-instalment, balloon, interest-only, bullet |
| DPD | Days Past Due — delinquency measure |
| NPL | Non-Performing Loan — typically 90+ DPD |
| Charge-off | Write the loan off the books as uncollectible |
| ECL (IFRS 9) | Expected Credit Loss — provisioning standard |
| Standard providers | Experian, Equifax, TransUnion (US/UK bureaus) |

---

## Core Concept

Origination is a funnel: application → fraud check → KYC → bureau pull → scorecard → decision. Each step has dropouts, and the funnel shape — application rate, approval rate, conversion to funded loan — is the lender's core product metric. Optimizing the funnel without compromising credit quality is the central tension of consumer lending.

The scorecard is the heart of the underwriting decision. Historically it has been logistic regression (transparent, well-understood), and increasingly it is gradient-boosted trees with explainability constraints layered on top (SHAP values, reason codes). Fair-lending regulation — ECOA in the US, the Equality Act in the UK — requires you to explain rejections to applicants, which constrains model choice in ways that pure ML teams sometimes miss.

Amortization math is fixed and well-known: given principal, rate, and term, the schedule is fully deterministic and can be computed in a closed form. The non-trivial part is what happens when borrowers pay late or early — different products have different rules for prepayment penalties, late fees, and re-amortization, and the rules must be encoded explicitly rather than improvised at runtime.

---

## Syntax & API

```csharp
public static IEnumerable<(int Period, Money Interest, Money Principal, Money Balance)>
    Amortize(Money principal, decimal annualRate, int months)
{
    var r = annualRate / 12m;
    var pmt = principal.Amount * r / (1 - (decimal)Math.Pow((double)(1 + r), -months));
    var balance = principal.Amount;
    for (var i = 1; i <= months; i++)
    {
        var interest = balance * r;
        var princ = pmt - interest;
        balance -= princ;
        yield return (i, new Money(interest, principal.Currency), new Money(princ, principal.Currency), new Money(balance, principal.Currency));
    }
}
```

---

## Common Patterns

```csharp
public sealed class ScoreResult(decimal Score, IReadOnlyList<string> ReasonCodes);

public interface IScorecard
{
    Task<ScoreResult> ScoreAsync(LoanApplication app, BureauData bureau, CancellationToken ct);
}
```

---

## Gotchas & Tips

- APR ≠ interest rate. APR includes fees; interest rate doesn't. Regulators care about APR.
- Day-count conventions matter (30/360, actual/365) — pick one per product and document.
- IFRS 9 ECL has three stages — performing, underperforming, credit-impaired — and the provision math differs per stage.
- Reject inference (modelling on people you didn't lend to) is a non-trivial bias control.

---

## See Also

- [[05 - Retail Banking Accounts and Transfers]]
- [[06 - Accounting Ledger and Double-Entry]]
- [[05 - Financial Compliance]]
