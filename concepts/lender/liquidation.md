# Liquidation

When a Borrower's health factor drops below 1, Liquidators can purchase the Borrower's delegated collateral by repaying their outstanding debt via a Dutch auction.&#x20;

## Mechanics

The liquidation process is as follows:

1. The liquidation process kicks off with a Liquidator calling the <kbd>openLiquidation</kbd> function&#x20;
2. After the liquidation has been initiated, the Borrower has a grace period of 12 hours to improve the health of the loan
   1. If the current Loan-to-Value exceeds the emergency liquidation threshold, the grace period is overridden, opening the liquidation window immediately.&#x20;
3. At the end of the grace period, Liquidators can call the <kbd>liquidate</kbd> function until the expiry of the liquidation period. Liquidations will expire in 3 days after the end of the grace period.&#x20;
4. A successful liquidation will execute a repayment of the borrowed asset, reducing the debt by the liquidation amount
5. The liquidated amount and the liquidation bonus is taken from the Underwriter's collateral via the Shared Security Network, transferring the collateral to the Liquidator
6. The liquidation window will close once the Borrower's health is recovered, i.e. health factor over 1

#### Liquidation Threshold

By default, the liquidation threshold is set to 80% LTV. If the current LTV exceeds the liquidation threshold, then a liquidation may be opened. An emergency liquidation mechanism is used to override the grace period if the health factor drops significantly. The emergency liquidation threshold is set to 90% LTV.&#x20;

#### Liquidation Expiry

All liquidations have an expiry window. The window ensures that once the position becomes healthy again, the Borrower will no longer be liquidatable. Hence, if the Borrower were to be liquidatable again, the Borrower will be ensured another grace period. If the Borrower's health factor is below 1 after the expiry, any Liquidator can initiate the liquidation process again.&#x20;

#### Target Health and Maximum Liquidatable Amount

Target health is a health factor threshold that defines the desired health level that liquidations should restore a Borrower's position to, currently set at 1.25. It acts as a "safety buffer" that liquidations aim to achieve.

The target health defines the maximum liquidatable amount. The maximum liquidatable amount is calculated as:&#x20;

<kbd>((Target Health \* Total Debt) - (Total Delegation \* Liquidation Threshold)) / ((Target Health - Liquidation Threshold) \* Asset Price)</kbd>

In other words, the goal is for the health factor, i.e.&#x20;

<kbd>Total Delegation \* Liquidation Threshold / (TotalDebt - Liquidated Amount)</kbd>

to be equal to the target health factor.

#### Liquidation Bonus (Dutch Auction)

The protocol implements a dynamic liquidation bonus system via a descending-price Dutch auction that provides incentives for Liquidators. The liquidation bonus rises linearly with time from the grace period until the maximum amount reaches the bonus cap. At expiry, Liquidators earn the maximum bonus which is set to 10%. If an emergency liquidation is triggered, Liquidators bypass the grace period, immediately earning the maximum bonus.&#x20;

Note that the bonus is only available when the total delegation exceeds total debt. The protocol prevents over-liquidating beyond available collateral.

For function signatures, parameters, and data structures, see the [Lender Contract Reference](../../developers/contracts/lender.md).
