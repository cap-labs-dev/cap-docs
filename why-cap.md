---
description: Verifiable Money, Verifiable Credit
---

# Why Cap?

Capital allocators in both legacy finance and DeFi encounter the principal-agent problem when allocating funds to yield opportunities: those that make decisions are not fully aligned with liquidity providers. This results in an unfavorable outcome for Lenders, with either risky or underperforming opportunities, opaque reporting, and uncertain recourse.

Cap fixes the root by changing the allocation mechanism: a credit marketplace where every loan is backed by onchain financial guarantees. Each loan has a dedicated Underwriter who puts their own capital behind the decision, making honest underwriting the dominant strategy. Allocation distributes across independent specialists rather than a single team, so rigor scales with the network.

Cap is designed by two guiding principles:&#x20;

* **Verification over trust:** Every guarantee is enforceable by smart contract. Collateral is onchain — it either exists or it doesn't. There is no NAV to massage, no bilateral agreement to misrepresent, no double-pledging discoverable only in bankruptcy court. Any Lender can verify their protection in real time.
* **Market driven:** Well-designed rewards and penalties minimize reliance on human judgement. Underwriters compete to back the best Borrowers. Borrowers compete for the lowest-cost credit lines. The market sets the price; the code enforces the rules.

### Why cUSD?&#x20;

Stablecoins are a better payment system compared to legacy financial systems — faster, safer, more transparent, and more accessible. However, as governments and financial institutions issue their own stablecoins, the ecosystem fractures into incompatible silos, negating the core value proposition of frictionless global value transfer.&#x20;

Money derives its value from network effects: the more users accept it, the more useful it becomes. For stablecoins to truly have "moneyness," solving fragmentation requires credibly neutral protocols that aggregate individual stablecoins.

cUSD offers singleness of money by aggregating compliant stablecoins into a unified reserve, creating an interconnected web of assets. It serves as a neutral bridge between the reserve assets, where users can redeem cUSD for any underlying asset. All logic is onchain, enabling fast, secure, and 24/7 settlement without preferential treatment of any reserve asset.&#x20;

As more institutions join Cap, the protocol serves as a settlement layer — an open standard for interoperability where every dollar is fungible regardless of who issued the stablecoin.

### Why stcUSD?&#x20;

Many of today's yield-bearing stablecoins are no different from tokenized hedge funds, relying on centralized teams to manage strategies. This model faces two major issues:

1. **Scalability:** A single team is limited by its resources and capacity. There are only so many strategies a team can run, and no strategy can infinitely scale. They quickly become obsolete as market conditions change.
2. **Safety:** Centralized management exposes users to uncovered risks that are opaque and subject to change at the discretion of the team. Users occupy the most junior position within protocol hierarchies, often lacking verifiable protection.

stcUSD addresses these challenges through financial guarantees and decentralized capital allocation:

1. **Verifiable downside protection:** Lenders are covered from the risk of yield generation via Underwriter collateral escrowed in Shared Security Networks. If a Borrower defaults, the Underwriter's collateral is liquidated before Lenders absorb any loss. All risk coverage is transparent and enforced by smart contracts, allowing users to verify protections onchain.&#x20;
2. **Perpetual competitive yield:** An open network of Borrowers competes to deliver the best yield, with their strategies underwritten by Underwriter capital. Smart contracts and game theory dynamically dictate capital allocation, ensuring stcUSD delivers competitive yield invariant to market conditions or scale.
3. **Modularity:** Each Underwriter-Borrower pair is isolated — one Underwriter's collateral backs exactly one Borrower. A single default cannot cascade to other Borrowers, removing the contagion risk inherent in pooled exposure models.
