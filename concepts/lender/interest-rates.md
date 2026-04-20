# Interest Rates

Cap's interest rate mechanism consists of benchmark rates, market rates, utilization rates, and restaker rates to ensure sustainable lending operations and proper compensation for risk-takers.

## Mechanics

The total interest that Operators pay can be summarized as:

* Minimum Rate + Utilization Rate + Restaker Rate

where Minimum rate is the maximum of the Benchmark Rate and the Market Rate

Each rate is defined as below:

1. **Benchmark Rate**: Minimum interest rates set by protocol, in fixed yearly rates
2. **Market Rate**: Dynamic rates fetched from external lending markets and oracles
3. **Utilization Rate**: Dynamic rates based on asset utilization. Calculated via a piecewise linear function and a rate multiplier
4. **Restaker Rate**: Fixed annual rate set paid by the Operator to the Delegator underwriting them

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Let us breakdown the interest formula:

#### 1. Minimum rate

The minimum rate represents a lower bound on the interest rates, and is determined by the greater value of the preconfigured benchmark rate and the market rate. The rationale for the minimum rate stems from the Fractional Reserves. Because Cap earns base yield from the underlying idle assets, Operators must be able to provide more than the base yield in order to borrow from Cap.

For instance, if the benchmark yield set by Cap is set at 5%, and the current strategy for the Fractional Reserve of USDC is AAVE v3 earning rate is 4.5%, the minimum rate would be 5%. The minimum rate ensures that Cap’s interest rates is greater than the base yield at all times.&#x20;

#### 2. Utilization Rate

The utilization rate is a function of a piecewise linear-kink model and a rate multiplier.&#x20;

The piecewise function allows interest rates to rise linearly along the first slope up to the "kink", or target utilization rate, set at 90%, after which rates increase rapidly along the second slope. This allows short term modulation of the rate where borrowing is disincentivized beyond the kink.&#x20;

The rate multiplier is determined by the deviation of the current utilization rate from the target rate.  If the current rate is far below the target rate, and remains there for a long period, then it must be the case that the current target rate is too high to incentivize borrowing. Based on the elapsed duration of the deviation and the degree of the difference between the current utilization rate and target, the multiplier will decrease the current rate, effectively shifting the piecewise function down. This control allows for a longer term modulation, optimizing Cap's interest rates in an autonomous way.

#### 3. Restaker Rate

The restaker rate is the fixed yearly rate that Operators pay to the Delegators that collateralize them,. The rate is determined as a bilateral agreement between the Operator and Delegator, and is unique per each pair.&#x20;

Putting the pieces together, when calculating the interest rate, the system does the following:

1. Fetch market rate from oracle
2. Compare with benchmark rate (minimum floor)
3. Use higher of market or benchmark rate
4. Add utilization rate on top

For function signatures, parameters, and data structures, see the [Lender Contract Reference](../../developers/contracts/lender.md) and [Oracle Contract Reference](../../developers/contracts/oracle.md).
