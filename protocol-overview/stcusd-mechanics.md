# stcUSD Mechanics

stcUSD is the yield-bearing stablecoin of Cap that enables users to earn rewards via a decentralized lending framework.

A network of operators can borrow from Cap's reserves based on their ability to generate yield over the hurdle rate of the asset.Borrowing in Cap is always over-collateralized: whitelisted Operator can permissionlessly access liquidity, provided that they receive sufficient collateral from Delegators. Under-collateralized loans trigger liquidation events, slashing Delegators from their Shared Security Networks. The slashed funds are redistributed back to the stablecoin holder, such that cUSD is backed 1:1 at all times.

Let us exemplify the mechanics of stcUSD.

## How does stcUSD work?

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption><p>Happy Path: Yield is distributed to all stakeholders</p></figcaption></figure>

1. **Minting and Staking**: Alice deposits \~$100 worth of $stable (a dollar-pegged asset) to the reserve to mint \~100 cUSD. She then stakes her cUSD in exchange for \~100 stcUSD.
2. **Idle Asset Rewards**: The idle $stable in the reserve automatically accrues rewards. Rewards can come directly from the underlying of $stable, or via integrated protocols (e.g. Aave, Morpho), determined programmatically depending on the rates.
3. **Restaker Delegations:**
   1. An operator identifies a yield opportunity exceeding Cap’s 8% hurdle rate\* for $stable. In order to borrow, the operator must first secure overcollateralized delegations from a willing restaker.
   2. A restaker runs due diligence on the operator and decides to delegate $stake. Restakers and operators can optionally enter into a bilateral agreement to negotiate terms such as loan tenor and fixed restaker rates (risk premium).
4. **Operator Borrowing**: The operator permissionlessly borrows $stable via Cap's smart contracts.

\*Note, 8% is an arbitrary number set as an example. The actual hurdle rate is a dynamic rate: for more information, refer to [Borrow Rates](stcusd-mechanics.md#borrow-rates)

There are two possible paths the flow can go from here.

### Happy Path: Competitive Yield

Let us first examine the happy path, where the operator successfully repays the loan and the value of $stake remains stable.

5. **Generate Yield**: The operator executes the strategy and generates yield. The operator successfully repays the loan and the associated interest in $stable.
6. **Yield Distribution**: Yield is distributed across all stakeholders. If the operator generates 15% yield on the principal, then 8% (hurdle rate) flows back to the stcUSD holders. The fixed restaker rate (assume 2%) is rewarded to the restakers, leaving the excess 5% as operator profit.

### Unhappy Path: Slashing and Redistribution

In Cap, the slashing logic is triggered under two objective conditions:

* Stake collateral value falls below safe thresholds
* The operator defaults on repayment of the loan

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption><p>Unhappy Path: Stablecoin Holders are protected against yield-generation risk</p></figcaption></figure>

When such an event happens, the path forks from step 5.

5. **Slashable Event**: A slashing event is triggered during an open loan. A dutch auction kicks off to clear the $stable debt of the operator.
6. **Liquidations**: Liquidators slash the restakers' $stake up to the point where the operator's position is restored to a healthy state. The slashed $stake is sold to the liquidators at a discounted price in exchange for $stable.
7. **Redistribution**: Th collected $stable is redistributed back to Cap’s reserve, ensuring stcUSD holders like Alice remain whole at all times.

## Borrow Rates

The borrow rate that the operator has to repay is a function of restaker rate and hurdle rate.

The **restaker rate** is the fixed annual premium negotiated with restakers.

The **hurdle rate** is a dynamic rate that is a function of market rate and utilization rate. The market rate is the minimum rate for the borrow rate, and acts as a benchmark yield from external protocols such as Aave. The utilization rate adjusts via a piecewise linear function, where the rate escalates sharply when reserve utilization exceeds target thresholds. Both market and utilization rate are applied per collateral asset.

This mechanism ensures that there is always liquidity available for withdrawals, while setting a competitive floor for the hurdle rate.
