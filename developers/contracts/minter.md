# Minter

The Minter is an abstract contract inherited by the Cap vault that handles fee calculation and routing for minting, burning, and redeeming cUSD. Dynamic fees are applied based on each backing asset's current allocation relative to its optimal ratio, incentivising deposits that balance the reserve and penalising those that over-concentrate it. Whitelisted addresses bypass all dynamic fees, and a flat redeem fee applies when burning cUSD proportionally across all backing assets.

## Mechanics

### Dynamic Mint/Burn Fees

Every cUSD mint or burn is priced using oracle-based valuations of the input asset and the cUSD token itself. After computing a pre-fee output amount, the contract calculates the **new ratio** — the asset's share of the total basket value after the swap — and runs it through a kinked piecewise-linear fee schedule.

**Mint fee zones** (based on post-mint asset ratio):

1. **Below or at `optimalRatio`**: Only `minMintFee` applies (a flat floor, capped at 5%, typically 10 bps).
2. **Between `optimalRatio` and `mintKinkRatio`**: Fee rises linearly from `minMintFee` toward `minMintFee + slope0`.
3. **Above `mintKinkRatio`**: Fee steepens further, adding `slope1` proportional to the excess above the kink.

**Burn fee zones** (based on post-burn asset ratio):

1. **At or above `optimalRatio`**: Zero fee — burning a well-allocated asset is always free.
2. **Between `burnKinkRatio` and `optimalRatio`**: Fee rises linearly using `slope0`.
3. **Below `burnKinkRatio`**: Fee steepens using `slope1` proportional to the shortfall below the kink.

All rates, ratios, and fees are expressed in **ray format** where `1e27 = 100%`. For example, a `minMintFee` of `1e24` is 0.1% (10 bps).

### Mint/Burn Process Flow

1. **Pre-fee amount**: `_amountOutBeforeFee` fetches oracle prices for the asset and cUSD, computes the raw output amount, and derives the post-swap allocation ratio.
2. **Whitelist check**: If `$.whitelist[user]` is `true`, the pre-fee amount is returned unchanged with zero fee.
3. **Fee application**: For non-whitelisted users, `_applyFeeSlopes` selects the correct zone from the kinked curve, computes the fee in ray arithmetic, and deducts it from the output.
4. **Return**: `(amountOut, fee)` is returned to the calling vault for execution.

### Redeem Process Flow

Redeeming burns cUSD in exchange for a **proportional slice of every backing asset** in the vault, at the current basket weights. This path avoids dynamic mint/burn fees entirely but applies a flat `redeemFee`.

1. A `shares` ratio is computed: `cUSD burned / total cUSD supply`, scaled to `1e33` precision.
2. For each asset in the vault, the user's proportional entitlement is `totalSupplies(asset) × shares / 1e33`.
3. If the user is whitelisted, `redeemFee` is set to zero; otherwise the flat fee is deducted from each asset amount.
4. Arrays of `amountsOut` and `fees` (one entry per asset) are returned.

### Deposit Caps

Each backing asset has an independent deposit cap (`depositCap[asset]`). Deposits that would push total supply beyond this cap revert. A cap of `0` means unlimited.

### Whitelisting

Whitelisted addresses receive fee-free minting, burning, and redeeming. This is intended for protocol-controlled addresses (e.g. the insurance fund, authorised integrators). The whitelist is managed by the access-controlled `setWhitelist` function.

---

## Architecture

### Core Functions

#### `getMintAmount(address _asset, uint256 _amountIn)`

```solidity
function getMintAmount(address _asset, uint256 _amountIn)
    external
    view
    returns (uint256 amountOut, uint256 fee);
```

Simulates a mint for `msg.sender`. Returns the cUSD amount received and the fee charged given the current basket state.

| Parameter   | Type      | Description                       |
|-------------|-----------|-----------------------------------|
| `_asset`    | `address` | The backing asset being deposited |
| `_amountIn` | `uint256` | Amount of `_asset` to deposit     |

**Returns**: `amountOut` — cUSD minted after fees; `fee` — cUSD equivalent of the fee deducted.

---

#### `getMintAmount(address _user, address _asset, uint256 _amountIn)`

```solidity
function getMintAmount(address _user, address _asset, uint256 _amountIn)
    external
    view
    returns (uint256 amountOut, uint256 fee);
```

Same as above but simulates for an arbitrary `_user` address. Use this overload when quoting on behalf of another address (e.g. checking whether a specific wallet is whitelisted).

| Parameter   | Type      | Description                                    |
|-------------|-----------|------------------------------------------------|
| `_user`     | `address` | Address whose whitelist status is applied      |
| `_asset`    | `address` | The backing asset being deposited              |
| `_amountIn` | `uint256` | Amount of `_asset` to deposit                  |

---

#### `getBurnAmount(address _asset, uint256 _amountIn)`

```solidity
function getBurnAmount(address _asset, uint256 _amountIn)
    external
    view
    returns (uint256 amountOut, uint256 fee);
```

Simulates burning `_amountIn` cUSD to receive `_asset`. Uses `msg.sender` for the whitelist check.

| Parameter   | Type      | Description                    |
|-------------|-----------|--------------------------------|
| `_asset`    | `address` | The backing asset to receive   |
| `_amountIn` | `uint256` | Amount of cUSD to burn         |

**Returns**: `amountOut` — asset amount received after fees; `fee` — fee in cUSD terms.

---

#### `getBurnAmount(address _user, address _asset, uint256 _amountIn)`

```solidity
function getBurnAmount(address _user, address _asset, uint256 _amountIn)
    external
    view
    returns (uint256 amountOut, uint256 fee);
```

Same as above but simulates for an arbitrary `_user`.

---

#### `getRedeemAmount(uint256 _amountIn)`

```solidity
function getRedeemAmount(uint256 _amountIn)
    external
    view
    returns (uint256[] memory amountsOut, uint256[] memory redeemFees);
```

Simulates redeeming `_amountIn` cUSD for a proportional share of every backing asset. Uses `msg.sender` for the whitelist check.

| Parameter   | Type      | Description              |
|-------------|-----------|--------------------------|
| `_amountIn` | `uint256` | Amount of cUSD to redeem |

**Returns**: `amountsOut` — array of asset amounts (one per vault asset, in vault asset order); `redeemFees` — flat fee deducted per asset.

---

#### `getRedeemAmount(address _user, uint256 _amountIn)`

```solidity
function getRedeemAmount(address _user, uint256 _amountIn)
    external
    view
    returns (uint256[] memory amountsOut, uint256[] memory redeemFees);
```

Same as above but simulates for an arbitrary `_user`.

---

#### `getFeeData(address _asset)`

```solidity
function getFeeData(address _asset) external view returns (FeeData memory feeData);
```

Returns the current fee schedule for `_asset`.

---

#### `setFeeData(address _asset, FeeData calldata _feeData)`

```solidity
function setFeeData(address _asset, FeeData calldata _feeData) external;
```

Updates the fee schedule for `_asset`. Access-controlled. Reverts with:

- `InvalidMinMintFee` if `minMintFee >= 0.05e27` (>= 5%)
- `InvalidMintKinkRatio` if `mintKinkRatio == 0` or `>= 1e27`
- `InvalidBurnKinkRatio` if `burnKinkRatio == 0` or `>= 1e27`
- `InvalidOptimalRatio` if `optimalRatio == 0`, `>= 1e27`, or equals either kink

---

#### `getRedeemFee()`

```solidity
function getRedeemFee() external view returns (uint256 redeemFee);
```

Returns the current flat redeem fee in ray format (`1e27 = 100%`).

---

#### `setRedeemFee(uint256 _redeemFee)`

```solidity
function setRedeemFee(uint256 _redeemFee) external;
```

Sets the flat redeem fee. Access-controlled. Emits `SetRedeemFee`.

---

#### `setWhitelist(address _user, bool _whitelisted)`

```solidity
function setWhitelist(address _user, bool _whitelisted) external;
```

Grants or revokes fee-free access for `_user`. Access-controlled. Emits `SetWhitelist`.

---

#### `setDepositCap(address _asset, uint256 _cap)`

```solidity
function setDepositCap(address _asset, uint256 _cap) external;
```

Sets the maximum total deposit allowed for `_asset`. `_cap = 0` means no limit. Access-controlled. Emits `SetDepositCap`.

---

#### `whitelisted(address _user)`

```solidity
function whitelisted(address _user) external view returns (bool isWhitelisted);
```

Returns `true` if `_user` is whitelisted (fee-exempt).

---

#### `depositCap(address _asset)`

```solidity
function depositCap(address _asset) external view returns (uint256 cap);
```

Returns the deposit cap for `_asset` in asset-native units.

---

### Data Structures

#### `MinterStorage`

Internal ERC-7201 namespaced storage layout for the Minter module.

```solidity
struct MinterStorage {
    address oracle;
    uint256 redeemFee;
    mapping(address => FeeData) fees;
    mapping(address => bool) whitelist;
    mapping(address => uint256) depositCap;
}
```

| Field        | Type                          | Description                                             |
|--------------|-------------------------------|---------------------------------------------------------|
| `oracle`     | `address`                     | Address of the price oracle used to value assets and cUSD |
| `redeemFee`  | `uint256`                     | Flat fee applied on redeem, in ray format (`1e27 = 100%`) |
| `fees`       | `mapping(address => FeeData)` | Fee schedule per backing asset                          |
| `whitelist`  | `mapping(address => bool)`    | Fee-exempt addresses                                    |
| `depositCap` | `mapping(address => uint256)` | Maximum asset deposit per backing asset, in asset-native units |

---

#### `FeeData`

Fee schedule for a single backing asset. All ratio and rate fields are in ray format (`1e27 = 100%`).

```solidity
struct FeeData {
    uint256 minMintFee;
    uint256 slope0;
    uint256 slope1;
    uint256 mintKinkRatio;
    uint256 burnKinkRatio;
    uint256 optimalRatio;
}
```

| Field           | Type      | Description                                                                                             |
|-----------------|-----------|---------------------------------------------------------------------------------------------------------|
| `minMintFee`    | `uint256` | Floor mint fee, always applied regardless of ratio. Max `0.05e27` (5%). Typical value: `1e24` (0.1%).  |
| `slope0`        | `uint256` | Rate of fee increase between `optimalRatio` and the kink ratio.                                         |
| `slope1`        | `uint256` | Rate of fee increase beyond the kink ratio — steeper than `slope0`.                                     |
| `mintKinkRatio` | `uint256` | Ratio above which `slope1` activates for mints. Must be in `(0, 1e27)`.                                 |
| `burnKinkRatio` | `uint256` | Ratio below which `slope1` activates for burns. Must be in `(0, 1e27)`.                                 |
| `optimalRatio`  | `uint256` | Target allocation ratio for the asset in the basket. Minting below this is cheap; burning above is free. |

---

### Events

| Event           | Parameters                        | Emitted when                            |
|-----------------|-----------------------------------|-----------------------------------------|
| `SetFeeData`    | `address asset, FeeData feeData`  | Fee schedule updated for an asset       |
| `SetRedeemFee`  | `uint256 redeemFee`               | Flat redeem fee updated                 |
| `SetWhitelist`  | `address user, bool whitelisted`  | Whitelist status changed for an address |
| `SetDepositCap` | `address asset, uint256 cap`      | Deposit cap updated for an asset        |

---

### Errors

| Error                  | Condition                                                                  |
|------------------------|----------------------------------------------------------------------------|
| `InvalidMinMintFee`    | `minMintFee >= 0.05e27` (5%)                                               |
| `InvalidMintKinkRatio` | `mintKinkRatio == 0` or `>= 1e27`                                          |
| `InvalidBurnKinkRatio` | `burnKinkRatio == 0` or `>= 1e27`                                          |
| `InvalidOptimalRatio`  | `optimalRatio == 0`, `>= 1e27`, or equals `mintKinkRatio`/`burnKinkRatio`  |

---

## Usage Examples

### 1. Quoting a mint before transacting

```solidity
interface IMinter {
    function getMintAmount(address _user, address _asset, uint256 _amountIn)
        external
        view
        returns (uint256 amountOut, uint256 fee);
}

address vault    = 0x...; // Cap vault proxy
address usdc     = 0x...;
address alice    = 0x...;
uint256 amountIn = 1000e6; // 1,000 USDC (6 decimals)

(uint256 cusdOut, uint256 feePaid) = IMinter(vault).getMintAmount(alice, usdc, amountIn);
// cusdOut  — cUSD minted after dynamic fee
// feePaid  — cUSD equivalent of the fee charged (goes to insurance fund)
```

### 2. Checking whitelist status before routing

```solidity
interface IMinter {
    function whitelisted(address _user) external view returns (bool);
    function getMintAmount(address _user, address _asset, uint256 _amountIn)
        external
        view
        returns (uint256 amountOut, uint256 fee);
}

function quoteMint(address vault, address user, address asset, uint256 amountIn)
    external
    view
    returns (uint256 amountOut, uint256 fee, bool isFeeExempt)
{
    isFeeExempt = IMinter(vault).whitelisted(user);
    (amountOut, fee) = IMinter(vault).getMintAmount(user, asset, amountIn);
}
```

### 3. Simulating a redeem across all basket assets

```solidity
interface IMinter {
    function getRedeemAmount(address _user, uint256 _amountIn)
        external
        view
        returns (uint256[] memory amountsOut, uint256[] memory redeemFees);
}

interface IVault {
    function assets() external view returns (address[] memory);
}

function previewRedeem(address vault, address user, uint256 cusdAmount)
    external
    view
    returns (address[] memory assetAddresses, uint256[] memory amounts, uint256[] memory fees)
{
    assetAddresses = IVault(vault).assets();
    (amounts, fees) = IMinter(vault).getRedeemAmount(user, cusdAmount);
    // amounts[i] and fees[i] correspond to assetAddresses[i]
}
```
