# Lender

The Lender module manages the borrowing, repayment, and liquidation processes for Borrowers.&#x20;

## Overview of Operations

1. [**Borrow/Repay**](borrow.md): Borrowers can borrow reserve assets against their delegated collateral
2. [**Liquidation**](liquidation.md): Multi-stage liquidation with grace periods and bonus incentives
3. [**Interest Rate Calculation**](interest-rates.md)**:** Dynamic protocol rates and fixed Borrower-specific rates

## Key Loan Parameters

* **Total Delegation:** Total amount of a Borrower's received collateral from Underwriters&#x20;
* **Total Liquidatable Collateral:** Total amount of collateral that can be liquidated from an Underwriter
* **Total Debt:** A Borrower's total debt denominated in USD. Interest accrues to Total Debt
* **Initial Loan-to-Value (LTV):** The maximum amount a Borrower can borrow relative to the Total Liquidatable Collateral. A 50% LTV implies that delegations must be at least 2x the size of the borrow.
* **Current LTV**: A Borrower's current Loan-to-Value ratio, calculated as&#x20;
  * (Total Debt / Total Delegation) \* 100%
* **Liquidation Threshold**: Threshold that determines when a Borrower's position becomes liquidatable in LTV ratio. By default, the threshold is set to 80%.
* **Health Factor**: Represents a Borrower's loan health
  * Calculated as (Total Delegation \* Liquidation Threshold) / Total Debt
  * Liquidations are triggered when health factor is below 1
* **Grace Period**: Period for Borrowers to recover health before liquidation can occur. Set to 12 hours
* **Expiry Period:** Period after which liquidation rights expire. Set to 3 days
* **Target health:** Target health factor that Liquidators aim to achieve when liquidating a Borrower. Set to 125%
* **Bonus Cap:** Maximum bonus for Liquidators, set to 10%.

## Debt Management

The protocol has two distinct types of interest:

1. **Vault Interest**: Interest paid to the Vault (stcUSD holders)
2. **Underwriting Premium**: Premium paid to Underwriters who provide collateral coverage

Debt in Cap is managed via Debt tokens. Debt tokens are non-transferrable ERC-20 tokens that track a Borrower's debt. Debt tokens are minted when a Borrower borrows, and burned when the debt is repaid.&#x20;

Interest automatically accrues to the debt token per asset, where the interest rate is calculated based on the [interest rate mechanism](interest-rates.md). Accrued interest is handled automatically via index-based scaling, inherited from the ScaledToken base class. &#x20;

For function signatures, parameters, and data structures, see the [Lender Contract Reference](../../developers/contracts/lender.md).
