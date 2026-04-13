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

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

The fee system operates across three distinct zones based on where the current ratio is:

1. `current ratio <= optimal ratio`: Minimum mint fee to mint, 0 fees for burn
2. `optimal ratio < current ratio < kinkRatio`: Linear fee increase using the first slope
3. `current ratio > kinkRatio`: Steep fee increase using the second slope

Price Oracles are used for fee calculations and amount determination. A minimum mint fee is placed to prevent manipulation around price oracles.

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

Redeem fees in Cap Protocol are simple percentage-based fees applied to proportional redemption operations. Same fees apply to all assets proportional to their asset allocation. Whitelisted users pay 0% redeem fee

**Process Flow for Redeem**:

1. **Fee Determination**: Sets redeem fee (0 if whitelisted)
2. **Share Calculation**: Calculates user's share of total supply
3. **Amount Calculation**: Calculates proportional amount for each asset
4. **Fee Application**: Applies redeem fee to each asset amount

### Whitelisting

Whitelisted users can bypass fees for larger scale operations. Whitelist management is restricted to authorized admins.

## Architecture

### Minter

#### Core Functions

**`getMintAmount`**: Calculates mint amount and fees for a given asset input

```solidity
function getMintAmount(address _asset, uint256 _amountIn) external view returns (uint256 amountOut, uint256 fee)
```

* `_asset`: Asset address to mint with
* `_amountIn`: Amount of asset to use for minting
* **Returns**: Amount of cUSD to mint and fee to be paid

**`getBurnAmount`**: Calculates burn amount and fees for a given cUSD input

```solidity
function getBurnAmount(address _asset, uint256 _amountIn) external view returns (uint256 amountOut, uint256 fee)
```

* `_asset`: Asset address to burn for
* `_amountIn`: Amount of cUSD to burn
* **Returns**: Amount of asset to receive and fee to be paid

**`getRedeemAmount`**: Calculates redemption amounts for multi-asset withdrawal

```solidity
function getRedeemAmount(uint256 _amountIn) external view returns (uint256[] memory amountsOut, uint256[] memory redeemFees)
```

* `_amountIn`: Amount of cUSD to redeem
* **Returns**: Amounts of each asset to receive and redeem fees

**Admin Functions**

**`setFeeData`**: Sets the fee parameters for an asset

```solidity
function setFeeData(address _asset, FeeData calldata _feeData) external checkAccess(this.setFeeData.selector)
```

* `_asset`: Asset address to configure
* `_feeData`: Complete fee configuration structure

**Parameters**:

* **minMintFee**: Minimum mint fee (maximum 5%)
* **slope0**: Fee slope below kink ratio
* **slope1**: Fee slope above kink ratio
* **mintKinkRatio**: Mint fee kink ratio threshold
* **burnKinkRatio**: Burn fee kink ratio threshold
* **optimalRatio**: Optimal allocation ratio

**`setRedeemFee`**: Sets the global redeem fee

```solidity
function setRedeemFee(uint256 _redeemFee) external checkAccess(this.setRedeemFee.selector)
```

* `_redeemFee`: Redeem fee percentage in ray format (1e27 = 100%)

**`setWhitelist`**: Sets whitelist status for a user

```solidity
function setWhitelist(address _user, bool _whitelisted) external checkAccess(this.setWhitelist.selector)
```

* `_user`: User address to modify
* `_whitelisted`: True to whitelist (exempt from fees), false to remove

**View Functions**

* **`whitelisted`**: Checks if a user is whitelisted

### **Minter Logic**

The MinterLogic library provides four main functions that handle the complete lifecycle of minting, burning, and redeeming operations:

1. **`amountOut`** - Main entry point for mint/burn calculations with fee application
2. **`redeemAmountOut`** - Handles proportional redemption calculations
3. **`_amountOutBeforeFee`** - Internal function for calculating the base amount and new asset ratio before applying fees
4. **`_applyFeeSlopes`** - Internal function for dynamic fee application based on the current ratio

### Data Structures

#### MinterStorage

```solidity
struct MinterStorage {
    address oracle;                           // Oracle address for pricing
    uint256 redeemFee;                        // Global redeem fee
    mapping(address => FeeData) fees;         // Fee data per asset
    mapping(address => bool) whitelist;       // Whitelist status per user
}
```

#### FeeData

```solidity
struct FeeData {
    uint256 minMintFee;        // Minimum mint fee (maximum 5%)
    uint256 slope0;            // Fee slope below kink ratio
    uint256 slope1;            // Fee slope above kink ratio
    uint256 mintKinkRatio;     // Mint fee kink ratio threshold
    uint256 burnKinkRatio;     // Burn fee kink ratio threshold
    uint256 optimalRatio;      // Optimal allocation ratio
}
```

#### FeeSlopeParams

```solidity
struct FeeSlopeParams {
    bool mint;                 // True if applying to mint, false if burn
    uint256 amount;            // Amount of asset to apply fee to
    uint256 ratio;             // Ratio of fee to apply
}
```

#### AmountOutParams

```solidity
struct AmountOutParams {
    bool mint;                 // True if minting, false if burning
    address asset;             // Asset address
    uint256 amount;            // Amount of asset to mint or burn
}
```

#### RedeemAmountOutParams

```solidity
struct RedeemAmountOutParams {
    uint256 amount;            // Amount of cap token to redeem
}
```
