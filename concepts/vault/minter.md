# Minter

The [Minter](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/vault/Minter.sol) contract handles fee calculation for the minting, burning and redeeming of cUSD, where dynamic fees are used to maintain the exposure of each backing asset to its optimal level. A balanced reserve composition reduces centralization of a particular asset, ensuring that the protocol is resilient to black swan events. Fees accrue to the treasury used for protocol safety.

## Mechanics

### Dynamic Mint/Burn Fees

Fees are dynamically adjusted according to the exposure of assets in the system. For each asset, there are two predefined piecewise linear functions that determine the mint/burn fees. Fees increase as they deviate from the optimal ratio, increasingly sharply beyond the kink ratio.

#### **Process Flow for Mint/Burn**:

1. **Pre-fee Calculation**: Calls `_amountOutBeforeFee` to get base amount and new ratio
2. **Whitelist Check**: Bypasses fees if user is whitelisted
3. **Fee Application**: Applies dynamic fees using `_applyFeeSlopes` if not whitelisted

#### Fee Calculations

<figure><img src="../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

The fee system operates across three distinct zones based on where the current ratio is:

1. `current ratio <= optimal ratio`: Minimum mint fee to mint, 0 fees for burn
2. `optimal ratio < current ratio < kinkRatio`: Linear fee increase using the first slope
3. `current ratio > kinkRatio`: Steep fee increase using the second slope

Price Oracles are used for fee calculations and amount determination. A minimum mint fee is placed to prevent manipulation around price oracles.&#x20;

The current minimum mint fee is set to 10bps, capped to 5% maximum.

#### Fee Parameters

| Parameter       | Description              | Validation       |
| --------------- | ------------------------ | ---------------- |
| `minMintFee`    | Base fee for minting     | ≤ 0.05e27 (5%)   |
| `optimalRatio`  | Target allocation ratio  | 0 < ratio < 1e27 |
| `mintKinkRatio` | Mint fee kink point      | 0 < ratio < 1e27 |
| `burnKinkRatio` | Burn fee kink point      | 0 < ratio < 1e27 |
| `slope0`        | First slope coefficient  |                  |
| `slope1`        | Second slope coefficient |                  |

### Redeem Fees

Redeem fees in Cap Protocol are simple percentage-based fees applied to proportional redemption operations. Same fees apply to all assets proportional to their asset allocation.  Whitelisted users pay 0% redeem fee

**Process Flow for Redeem**:

1. **Fee Determination**: Sets redeem fee (0 if whitelisted)
2. **Share Calculation**: Calculates user's share of total supply
3. **Amount Calculation**: Calculates proportional amount for each asset
4. **Fee Application**: Applies redeem fee to each asset amount

### Whitelisting

Whitelisted users can bypass fees for larger scale operations. Whitelist management is restricted to authorized admins.

For function signatures, parameters, and data structures, see the [Minter Contract Reference](../../developers/contracts/minter.md).

