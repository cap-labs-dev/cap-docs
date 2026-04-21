---
description: Verifiable Credit
---

# Why Cap?

## The Problem

\
Capital allocators in both legacy finance and DeFi encounter the principal-agent problem when allocating funds to yield opportunities: those that make decisions are not fully aligned with liquidity providers. This misalignment of incentives results in an unfavorable outcome for Lenders, with either risky or underperforming opportunities, opaque reporting, and uncertain recourse.

More concretely, the structural problems in the credit space today are the following:<br>

1. **Overfitting for growth over quality:** Originators who do not hold the loans they make have little direct reason to underwrite carefully. Throughput and AUM growth are rewarded more directly than credit quality.
2. **Scalability:** Underwriting and origination bottleneck on a small set of institutional decision-makers. With finite capacity, they either overallocate to existing borrowers or originate more than they can underwrite rigorously.
3. **Opacity**: Portfolios are priced by the same managers who originated the loans. Collateral is tracked through borrower self-reporting and cannot be verified real time. Misreporting and double-pledging tend to surface only after the damage is done.

## Cap's Solution

Cap fixes the root by changing the allocation mechanism: a credit marketplace where every loan is backed by onchain financial guarantees. Each loan has a dedicated Underwriter who puts their own capital behind the decision, making honest underwriting the dominant strategy. Allocation distributes across independent specialists rather than a single team, so rigor scales with the network.

Cap is designed by two guiding principles:

* **Verification over trust:** Every guarantee is enforceable by smart contract. Collateral is onchain so that any Lender can verify their protection in real time.
* **Market driven:** Well-designed rewards and penalties minimize reliance on human judgement. Underwriters compete to back the best Borrowers. Borrowers compete for the lowest-cost credit lines. The market sets the price; the code enforces the rules.

Together, these principles produce:

1. **Safe yield.** Lenders of any size access institutional credit yield, protected by escrowed, overcollateralized guarantees. Because each Underwriter–Borrower pair is isolated, a default in one position cannot cascade to others.
2. **Scalable yield:** An open network of Borrowers competes to deliver the best yield. Game theory dynamically dictates capital allocation, ensuring stcUSD delivers competitive yield invariant to market conditions or scale.
3. **Verifiability.** Every guarantee is enforceable by smart contract. A Underwriter's collateral is onchain and backs exactly one Borrower. Any Lender can verify their protection on the blockchain in real time.
4. **Global and open.** Underwriters and Borrowers can participate across geographies on public blockchain rails.
