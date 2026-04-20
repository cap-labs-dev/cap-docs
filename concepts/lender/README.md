# Lender

The Lender module manages the borrowing, repayment, and liquidation processes for Operators.&#x20;

## Overview of Operations

1. [**Borrow/Repay**](borrow.md): Operators can borrow reserve assets against their delegated collateral
2. [**Liquidation**](liquidation.md): Multi-stage liquidation with grace periods and bonus incentives
3. [**Interest Rate Calculation**](interest-rates.md)**:** Dynamic protocol rates and fixed Operator specific rates

## Key Loan Parameters

* **Total Delegation:** Total amount of an Operator's received collateral from Delegators&#x20;
* **Total Slashable Collateral:** Total amount of collateral that can be slashed from a Delegator
* **Total Debt:** An Operator's total debt denominated in USD. Interest accrues to Total Debt
* **Initial Loan-to-Value (LTV):** The maximum amount an Operator can borrow relative to the Total Slashable Collateral. A 50% LTV implies that delegations must be at least 2x the size of the borrow.
* **Current LTV**: An Operator's current Loan-to-Value ratio, calculated as&#x20;
  * (Total Debt / Total Delegation) \* 100%
* **Liquidation Threshold**: Threshold that determines when an Operator's position becomes slashable in LTV ratio. By default, the threshold is set to 80%.
* **Health Factor**: Represents an Operator's loan health
  * Calculated as (Total Delegation \* Liquidation Threshold) / Total Debt
  * Liquidations are triggered when Health is below 1
* **Grace Period**: Period for Operators to recover the health before liquidation can occur. Set to 12 hours
* **Expiry Period:** Period after which liquidation rights expire. Set to 3 days
* **Target health:** Target health ratio that liquidators aim to achieve when slashing an Operator. Set to 125%
* **Bonus Cap:** Maximum bonus for liquidators, set to 10%.

## Debt Management

The protocol has two distinct types of interest:

1. **Vault Interest**: Interest paid to the Vault (stcUSD holders)
2. **Restaker Interest**: Interest paid to Delegators who provide collateral coverage

Debt in Cap is managed via Debt tokens. Debt tokens are non-transferrable ERC-20 tokens that track an Operator's debt. Debt tokens are minted when an Operator borrows, and burned when the debt is repayed.&#x20;

Interest automatically accrues to the debt token per asset, where the interest rate is calculated based on the [interest rate mechanism](interest-rates.md). Accrued interest is handled automatically via index-based scaling, inherited from the ScaledToken base class. &#x20;

For function signatures, parameters, and data structures, see the [Lender Contract Reference](../../developers/contracts/lender.md).

