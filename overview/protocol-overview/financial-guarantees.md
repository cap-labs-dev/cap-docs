---
description: On why Cap makes sense
---

# Financial Guarantees

A **financial guarantee** is a binding agreement where an Underwriter guarantees repayment to the Lender, with the Borrower in turn guaranteeing repayment to both. If the Borrower defaults, the Underwriter covers the loss and retains recourse against the Borrower. This distinguishes financial guarantees from instruments like credit default swaps, where no such right of recovery exists.

Cap splits the guarantee across two layers:

* **Onchain**, the Underwriter escrows collateral to back a specific loan and receives the underwriting premium via smart contract. Default triggers immediate, automated liquidation of that collateral to make the Lender whole.
* **Offchain**, Underwriter and Borrower enter a legal agreement that provides the Underwriter's right of recovery against the Borrower after liquidation.

Each Underwriter has full agency over risk-reward. They assess Borrower credit, set LTVs per Borrower, and negotiate the premium bilaterally. Every guarantee is priced by the Underwriter who stands behind it — not by pooled governance or protocol-wide risk parameters.

The mechanism turns idle alternative assets — BTC, ETH, and tokenized RWAs (commodities, alternative currencies, stocks) — into productive underwriting capital. By backing USD-denominated loans with their holdings, Underwriters capture the yield differential between USD and their own asset class without selling the underlying.

**Excess Value**

To understand how Cap can generate value via financial guarantees, let us follow the cashflows on a single loan.

A Borrower pays the full unsecured USD borrow rate. That payment must cover two things: the Lender's required return (the USD risk-free rate) and the Underwriter's required return (the unsecured borrow rate for the alternative asset, since that is the opportunity cost of locking it up).

The surplus between these is **Excess Value**:

`Excess Value = USD unsecured borrow rate − USD risk-free rate − alternative asset unsecured borrow rate`

Because the escrowed collateral is unfunded — it backs the loan without being consumed — it continues to earn its native yield in parallel. Fiat reserves sweep into money-market funds; crypto assets stake on their L1. This adds a second layer of yield:

`Excess Value = USD unsecured borrow rate + alternative asset native yield − USD risk-free rate − alternative asset unsecured borrow rate`

How Excess Value is split among participants is **dynamically determined** based on market forces. Lenders earn the floating hurdle rate based on utilization rate. Each Underwriter earns the premium they individually charge the Borrower.

**Worked Example: ETH**

Consider a $100M USD loan backed by $200M of staked ETH collateral (200% overcollateralization) with hypothetical rates set below.&#x20;

| Parameter                  | Value |
| -------------------------- | ----- |
| USD unsecured borrow rate  | 10%   |
| USD risk-free rate         | 4%    |
| ETH unsecured lending rate | 4%    |
| ETH native (PoS) yield     | 3%    |

**Flow of cashflows**

```
$10M from Borrower
  ├─→ $4M  Lenders' minimum        (risk-free rate on $100M)
  ├─→ $2M  Underwriters' minimum   (4% target on $200M − 3% native PoS)
  └─→ $4M  Excess Value            (available for distribution)
```

Lenders earn the floating hurdle rate above the risk-free floor, Underwriters earn their negotiated premium above the 4% minimum, and the Excess Value is distributed by market pricing relative to the asset premium.

***
