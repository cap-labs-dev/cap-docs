# stcUSD Mechanics

stcUSD is the yield-bearing stablecoin of Cap that enables users to earn rewards via a decentralized lending framework.&#x20;

A network of operators can borrow from Cap's reserves based on their ability to generate yield over the hurdle rate of the asset.Borrowing in Cap is always over-collateralized: whitelisted Operator can permissionlessly access liquidity, provided that they receive sufficient collateral from Delegators. Under-collateralized loans trigger liquidation events, slashing Delegators from their Shared Security Networks. The slashed funds are redistributed back to the stablecoin holder, such that cUSD is backed 1:1 at all times.

Let us exemplify the mechanics of stcUSD.

## How does stcUSD work?

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption><p>Happy Path: Yield is distributed to all stakeholders</p></figcaption></figure>

1. **Minting and Staking**: Alice deposits \~$100 worth of $stable (a dollar-pegged asset) to the reserve to mint \~100 cUSD. She then stakes her cUSD in exchange for \~100 stcUSD.
2. **Idle Asset Rewards**: The idle $stable in the reserve automatically accrues rewards. Rewards can come directly from the underlying of $stable, or via integrated protocols (e.g. Aave, Morpho), determined programmatically depending on the rates.
3. **Restaker Delegations:**
   1. An operator identifies a yield opportunity exceeding Cap’s 8% hurdle rate\* for $stable. In order to borrow, the operator must first secure overcollateralized delegations from a willing restaker.&#x20;
   2. A restaker runs due diligence on the operator and decides to delegate $stake. Restakers and operators can optionally enter into a bilateral agreement to negotiate terms such as loan tenor and fixed restaker rates (risk premium).&#x20;
4. **Operator Borrowing**: The operator permissionlessly borrows $stable via Cap's smart contracts.

\*Note, 8% is an arbitrary number set as an example. The actual hurdle rate is a dynamic rate: for more information, refer to [Borrow Rates](stcusd-mechanics.md#borrow-rates)

There are two possible paths the flow can go from here.&#x20;

### Happy Path: Competitive Yield

Let us first examine the happy path, where the operator successfully repays the loan and the value of $stake remains stable.

5. **Generate Yield**: The operator executes the strategy and generates yield. The operator successfully repays the loan and the associated interest in $stable.
6. **Yield Distribution**: Yield is distributed across all stakeholders.  If the operator generates 15% yield on the principal, then 8% (hurdle rate) flows back to the stcUSD holders. The fixed restaker rate (assume 2%) is rewarded to the restakers, leaving the excess 5% as operator profit.&#x20;

### Unhappy Path: Slashing and Redistribution

In Cap, the slashing logic is triggered by one objective condition: the operator’s health factor (ratio of collateral to debt) drops below the liquidation threshold. This can happen if the value of the restaker’s delegated collateral falls or the operator’s debt grows beyond safe levels.

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption><p>Unhappy Path: Stablecoin Holders are protected against yield-generation risk</p></figcaption></figure>

When such an event happens, the path forks from step 5.

5. **Slashable Event**: The operator’s health factor drops below the liquidation threshold. After a grace period, a liquidation window opens permissionlessly.
6. **Liquidations**: Liquidators repay a portion of the operator’s debt in exchange for the restakers’ $stake at a bonus. The bonus grows over time to incentivize faster liquidation. Liquidators slash up to the amount needed to restore the operator’s position to a healthy state.
7. **Redistribution**: The collected $stable is redistributed back to Cap’s reserve, ensuring stcUSD holders like Alice remain whole at all times.

## Borrow Rates

The borrow rate that the operator has to repay is a function of restaker rate and hurdle rate.&#x20;

The **restaker rate** is the fixed annual premium negotiated with restakers.

The **hurdle rate** is a dynamic rate composed of a minimum rate and a utilization rate:

* **Minimum rate**: The greater of the benchmark rate (a protocol-set floor) and the market rate (current borrow rate from external protocols such as Aave). This ensures Cap's lending rates always exceed the base yield earned from idle capital in Fractional Reserves.
* **Utilization rate**: Adjusts via a piecewise linear function, escalating sharply when reserve utilization exceeds target thresholds.

Both rates are applied per reserve asset. This mechanism ensures that there is always liquidity available for withdrawals, while setting a competitive floor for the hurdle rate. For more detail, see [Interest Rates](../concepts/lender/interest-rates.md).
