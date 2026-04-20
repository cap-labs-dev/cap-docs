# FAQs

## Borrower Control and Restrictions

**Q: Do Borrowers have full control over collateral once disbursed?**

No. The Borrower set will be comprised of regulated financial institutions that enter into legal agreements with Underwriters to restrict permissible strategies.&#x20;

**Q. How does the protocol ensure there are no risky strategies?**

Underwriters bear direct risk for their delegation choices. By underwriting specific Borrowers, Underwriters incentivize due diligence on strategy viability.

**Q: Are there caps on collateral lent to Borrowers?**

Yes. Delegation limits and nominal exposure thresholds prevent concentration risk.

## Loan Security

**Q: Does the protocol impose minimum overcollateralization requirements on Underwriters?**&#x20;

Yes, the protocol checks the collateralization ratio before the Borrower can borrow. The overcollateralization ratio is different for every collateral asset and is set conservatively to prevent any unexpected liquidations.

**Q: Which assets are accepted as Underwriter collateral?**

Only blue-chip cryptocurrencies: ETH, wBTC, Liquid Staking Tokens (LSTs), and stablecoins.

**Q: What happens if Underwriters withdraw their delegation mid-loan?**&#x20;

Underwriters and Borrowers should have a prior agreement when Underwriters intend to remove delegation for position unwinding. Unilateral withdrawals will trigger liquidation. Note that there is a built-in buffer period in Shared Security Networks to mitigate an accidental withdrawal.

## Shared Security Network

**Q. How is Liquidation and Redistribution used in Cap?**

Malicious or undercollateralized Borrowers are autonomously liquidated, where Underwriter delegations are redistributed back to the Lenders.

**Q. How does Cap differ from other restaking protocols?**

Cap replaces passive Proof-of-Stake rewards with productivity-based incentives:

* Underwriters earn premiums tied to Borrower performance.
* Borrowers retain surplus yield, fostering competitive strategy innovation.

Cap takes an innovative approach of rewarding Underwriters on a Borrower basis, where counterparty risk is established one-to-one with the Borrower they are underwriting. This architecture allows Borrowers to take more agency and be rewarded for their productivity.&#x20;

For details, please check out our [Productivity Based Incentives](https://mirror.xyz/0x83c21bb4Bf0EC116f5a1945AaeF847Fe3b321B32/Xg7gCAqgmcxgXhuwaWXVJDSCEIMn3_dewNWAie2Tn_k) article.
