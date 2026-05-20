# Protocol Overview

<figure><img src="../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

### Protocol Overview

Cap is a credit marketplace where the functions of underwriting, borrowing, and lending are separated and enforced by code. Underwriters escrow collateral to backstop Borrowers. Those Borrowers gain access to credit lines funded by Lenders depositing dollar denominated assets. The Underwriter's escrowed collateral is the first-loss buffer: if a Borrower defaults, the collateral is liquidated before Lenders absorb any loss. Every guarantee, every collateral position, and every Borrower exposure can be verified onchain.

### Protocol Actors

* **Lenders** receive cUSD in exchange for depositing whitelisted assets to the collateral pool. cUSD can be staked for stcUSD to earn auto-compounding yield on institutional loans, with principal protected by Underwriter collateral.
* **Borrowers** draw reserve assets against Underwriter collateral to execute yield strategies. When repaying, they pay the hurdle rate: an aggregate of underwriting premium and borrow rate.
* **Underwriters** escrow collateral to back a specific Borrower, earning a fixed underwriting premium they set per loan. Underwriter delegations are used to protect Lenders against default risk.
* **Liquidators** liquidate Underwriter collateral via permissionless reverse Dutch auction when a Borrower's health factor falls below threshold.

### **Aligning incentives**

* **Lenders** earn yield regardless of market condition and size. More importantly, their principal is protected by verifiable financial guarantees.
* **Borrowers** can source on demand capital with no cost basis, retaining any surplus yield after hurdle rate fees. Furthermore, borrow terms are flexible as Borrowers are able to negotiate with their Underwriter.
* **Underwriters** earn premiums in exchange for their delegations. They have full autonomy to charge risk premium per Borrower, paid out in USD denominated assets.
* **Liquidators** earn liquidation bonuses. Liquidators can earn up fees up to the predetermined bonus cap.

### Flow of Funds

1. **Deposit.** Lenders deposit whitelisted reserve assets (stablecoins, tokenized MMFs) to the Cap Reserve, minting cUSD. Staked cUSD (stcUSD) accrues yield.
2. **Delegate.** Underwriters escrow collateral to underwrite a specific Borrower for an underwriting premium.
3. **Borrow.** Borrowers draw reserve assets liquidity against that delegation to execute a yield strategy.
4. **Repay.** Borrowers repay principal and interest. Underwriters earn an underwriting premium, Lenders earn yield, Borrowers retain the surplus.
5. **Liquidate.** If a Borrower acts maliciously or collateral drops significantly in value, Underwriter collateral is liquidated and redistributed to the Reserve — cUSD remains fully backed at all times.
