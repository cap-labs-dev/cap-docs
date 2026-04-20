# stcUSD Mechanics

stcUSD is the yield-bearing stablecoin of Cap that enables users to earn rewards via a decentralized credit marketplace.&#x20;

A network of Borrowers can draw from Cap's reserves based on their ability to generate yield over the hurdle rate of the asset. Borrowing in Cap is always overcollateralized: whitelisted Borrowers can permissionlessly access liquidity, provided that they receive sufficient collateral from Underwriters. Undercollateralized loans trigger liquidation events, liquidating Underwriter collateral from their Shared Security Networks. The liquidated funds are redistributed back to the stablecoin holder, such that cUSD is backed 1:1 at all times.

Let us exemplify the mechanics of stcUSD.

## How does stcUSD work?

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption><p>Happy Path: Yield is distributed to all stakeholders</p></figcaption></figure>

1. **Minting and Staking**: Alice deposits \~$100 worth of $stable (a dollar-pegged asset) to the reserve to mint \~100 cUSD. She then stakes her cUSD in exchange for \~100 stcUSD.
2. **Idle Asset Rewards**: The idle $stable in the reserve automatically accrues rewards. Rewards can come directly from the underlying of $stable, or via integrated protocols (e.g. Aave, Morpho), determined programmatically depending on the rates.
3. **Underwriter Delegations:**
   1. A Borrower identifies a yield opportunity exceeding Cap's 8% hurdle rate\* for $stable. In order to borrow, the Borrower must first secure overcollateralized delegations from a willing Underwriter.&#x20;
   2. An Underwriter runs due diligence on the Borrower and decides to delegate $stake. Underwriters and Borrowers can optionally enter into a bilateral agreement to negotiate terms such as loan tenor and fixed underwriting premiums (risk premium).&#x20;
4. **Borrower Draws**: The Borrower permissionlessly borrows $stable via Cap's smart contracts.

\*Note, 8% is an arbitrary number set as an example. The actual hurdle rate is a dynamic rate: for more information, refer to [Borrow Rates](stcusd-mechanics.md#borrow-rates)

There are two possible paths the flow can go from here.&#x20;

### Happy Path: Competitive Yield

Let us first examine the happy path, where the Borrower successfully repays the loan and the value of $stake remains stable.

5. **Generate Yield**: The Borrower executes the strategy and generates yield. The Borrower successfully repays the loan and the associated interest in $stable.
6. **Yield Distribution**: Yield is distributed across all stakeholders. If the Borrower generates 15% yield on the principal, then 8% (hurdle rate) flows back to the stcUSD holders. The fixed underwriting premium (assume 2%) is rewarded to the Underwriters, leaving the excess 5% as Borrower profit.&#x20;

### Unhappy Path: Liquidation and Redistribution

In Cap, the liquidation logic is triggered by one objective condition: the Borrower's health factor (ratio of collateral to debt) drops below the liquidation threshold. This can happen if the value of the Underwriter's delegated collateral falls or the Borrower's debt grows beyond safe levels.

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption><p>Unhappy Path: Stablecoin Holders are protected against yield-generation risk</p></figcaption></figure>

When such an event happens, the path forks from step 5.

5. **Liquidation Event**: The Borrower's health factor drops below the liquidation threshold. After a grace period, a liquidation window opens permissionlessly.
6. **Dutch Auction**: Underwriter collateral is auctioned via a descending-price Dutch auction. Liquidators repay a portion of the Borrower's debt in exchange for the Underwriters' $stake at a bonus. The bonus grows over time to incentivize faster liquidation. Liquidators liquidate up to the amount needed to restore the Borrower's position to a healthy state.
7. **Redistribution**: The collected $stable is redistributed back to Cap's reserve, ensuring stcUSD holders like Alice remain whole at all times.

## Borrow Rates

The borrow rate that the Borrower has to repay is a function of underwriting premium and hurdle rate.&#x20;

The **underwriting premium** is the fixed annual premium negotiated with Underwriters.

The **hurdle rate** is a dynamic rate composed of a minimum rate and a utilization rate:

* **Minimum rate**: The greater of the benchmark rate (a protocol-set floor) and the market rate (current borrow rate from external protocols such as Aave). This ensures Cap's lending rates always exceed the base yield earned from idle capital in Fractional Reserves.
* **Utilization rate**: Adjusts via a piecewise linear function, escalating sharply when reserve utilization exceeds target thresholds.

Both rates are applied per reserve asset. This mechanism ensures that there is always liquidity available for withdrawals, while setting a competitive floor for the hurdle rate. For more detail, see [Interest Rates](../concepts/lender/interest-rates.md).
