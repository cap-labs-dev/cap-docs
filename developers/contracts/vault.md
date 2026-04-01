# Vault

The Vault is the core abstract contract inherited by every cUSD deployment. It holds the backing assets deposited by cUSD minters and lent out to whitelisted Operators, tracking total supplies and borrows per asset. Mint/burn/redeem operations flow through the Minter for fee calculation and through the FractionalReserve layer for liquidity management — the Vault itself only tracks principal amounts, leaving interest calculations to the Lender.

## Mechanics

### Minting cUSD

Depositing a backing asset in exchange for cUSD:

1. **Deposit cap check**: `_amountIn` is silently capped to the remaining deposit capacity for the asset.
2. **Fee calculation**: The Minter computes `(amountOut, fee)` based on the asset's current allocation ratio vs. its optimal ratio.
3. **Slippage / deadline check**: Reverts if `amountOut < _minAmountOut` or the deadline has passed.
4. **Transfer in**: The asset is pulled from `msg.sender` and `totalSupplies[asset]` is incremented.
5. **Mint**: `amountOut` cUSD is minted to `_receiver`; if a fee applies, an additional `fee` cUSD is minted to the insurance fund.

### Burning cUSD

Exchanging cUSD for a single backing asset:

1. **Fee calculation**: The Minter computes the post-fee asset output and fee amount.
2. **cUSD burn**: `_amountIn` cUSD is burned from `msg.sender`.
3. **Divest**: The FractionalReserve layer divests `amountOut + fee` of the asset from the active yield strategy if needed to cover the withdrawal.
4. **Transfer out**: `amountOut` is sent to `_receiver`; `fee` is sent to the insurance fund. `totalSupplies[asset]` is decremented.

### Redeeming cUSD

Proportional withdrawal across every backing asset:

1. **Amount calculation**: The Minter computes proportional `amountsOut` and `fees` for each asset in the vault.
2. **cUSD burn**: `_amountIn` cUSD is burned from `msg.sender`.
3. **Divest many**: The FractionalReserve layer divests the required amounts across all assets.
4. **Transfer out**: For each asset, `amountsOut[i]` is sent to `_receiver` and `fees[i]` to the insurance fund.

### Borrowing and Repaying

These paths are access-controlled and called only by the Lender contract:

- **Borrow**: Divests the asset from the yield strategy if needed, decrements available balance, transfers to `_receiver`, increments `totalBorrows[asset]`.
- **Repay**: Accepts the asset back from the Lender via `transferFrom`, decrements `totalBorrows[asset]`.

### Utilization Tracking

The vault tracks a **cumulative utilization index** per asset — a time-weighted integral of the utilization ratio. This is consumed by the RateOracle to compute borrow interest rates.

```
utilizationIndex += utilization * (block.timestamp - lastUpdate)
utilization = totalBorrows / totalSupplies  (ray format, 1e27 = 100%)
```

The index is updated on every mint, burn, borrow, and repay via the `updateIndex` modifier.

---

## Architecture

### Core Functions

#### `mint(address _asset, uint256 _amountIn, uint256 _minAmountOut, address _receiver, uint256 _deadline)`

```solidity
function mint(
    address _asset,
    uint256 _amountIn,
    uint256 _minAmountOut,
    address _receiver,
    uint256 _deadline
) external returns (uint256 amountOut);
```

Deposit `_asset` and receive cUSD. Requires ERC20 approval from `msg.sender` to the vault. `_amountIn` is silently reduced to the remaining deposit cap if exceeded.

| Parameter      | Type      | Description                                       |
|----------------|-----------|---------------------------------------------------|
| `_asset`       | `address` | Whitelisted backing asset to deposit              |
| `_amountIn`    | `uint256` | Amount of asset to deposit                        |
| `_minAmountOut`| `uint256` | Minimum cUSD to receive (slippage protection)     |
| `_receiver`    | `address` | Recipient of the minted cUSD                      |
| `_deadline`    | `uint256` | Unix timestamp after which the tx reverts         |

**Returns**: `amountOut` — cUSD minted to `_receiver` (excludes fee).

---

#### `burn(address _asset, uint256 _amountIn, uint256 _minAmountOut, address _receiver, uint256 _deadline)`

```solidity
function burn(
    address _asset,
    uint256 _amountIn,
    uint256 _minAmountOut,
    address _receiver,
    uint256 _deadline
) external returns (uint256 amountOut);
```

Burn `_amountIn` cUSD and receive `_asset`. The caller must hold the cUSD.

| Parameter      | Type      | Description                                    |
|----------------|-----------|------------------------------------------------|
| `_asset`       | `address` | Backing asset to withdraw                      |
| `_amountIn`    | `uint256` | Amount of cUSD to burn                         |
| `_minAmountOut`| `uint256` | Minimum asset to receive (slippage protection) |
| `_receiver`    | `address` | Recipient of the asset                         |
| `_deadline`    | `uint256` | Unix timestamp after which the tx reverts      |

**Returns**: `amountOut` — asset sent to `_receiver` (excludes fee).

---

#### `redeem(uint256 _amountIn, uint256[] calldata _minAmountsOut, address _receiver, uint256 _deadline)`

```solidity
function redeem(
    uint256 _amountIn,
    uint256[] calldata _minAmountsOut,
    address _receiver,
    uint256 _deadline
) external returns (uint256[] memory amountsOut);
```

Burn `_amountIn` cUSD and receive a proportional share of every backing asset. `_minAmountsOut` must have one entry per vault asset in the same order as `assets()`.

**Returns**: `amountsOut` — asset amounts sent to `_receiver` (one per vault asset).

---

#### `borrow(address _asset, uint256 _amount, address _receiver)`

```solidity
function borrow(address _asset, uint256 _amount, address _receiver) external;
```

Access-controlled. Called by the Lender to transfer assets to an Operator. Reverts with `InsufficientReserves` if available balance is too low.

---

#### `repay(address _asset, uint256 _amount)`

```solidity
function repay(address _asset, uint256 _amount) external;
```

Access-controlled. Called by the Lender when an Operator repays. Pulls `_amount` from `msg.sender` via `transferFrom`.

---

#### `availableBalance(address _asset)`

```solidity
function availableBalance(address _asset) external view returns (uint256 amount);
```

Returns `totalSupplies[_asset] - totalBorrows[_asset]` — the amount currently available to borrow or withdraw.

---

#### `utilization(address _asset)`

```solidity
function utilization(address _asset) external view returns (uint256 ratio);
```

Returns the current utilization ratio in ray format (`1e27 = 100%`). For example, `5e26` = 50% utilization.

---

#### `currentUtilizationIndex(address _asset)`

```solidity
function currentUtilizationIndex(address _asset) external view returns (uint256 index);
```

Returns the up-to-date cumulative utilization index including the current unsettled period. Used by the RateOracle for interest rate calculations.

---

#### `getRemainingMintCapacity(address _asset)`

```solidity
function getRemainingMintCapacity(address _asset) external view returns (uint256 remainingMintCapacity);
```

Returns how much more of `_asset` can be deposited before hitting the deposit cap. Returns 0 if the cap is already reached.

---

#### Admin Functions

| Function | Description |
|----------|-------------|
| `addAsset(address _asset)` | Add a new backing asset to the vault |
| `removeAsset(address _asset)` | Remove an asset (only if `totalSupplies == 0`) |
| `pauseAsset(address _asset)` | Pause minting, burning, and borrowing for a specific asset |
| `unpauseAsset(address _asset)` | Resume operations for a paused asset |
| `pauseProtocol()` | Pause all vault operations globally |
| `unpauseProtocol()` | Resume all vault operations |
| `setInsuranceFund(address)` | Update the insurance fund address |
| `rescueERC20(address _asset, address _receiver)` | Recover a non-vault, non-strategy token sent to the contract by mistake |

---

### Data Structures

#### `VaultStorage`

ERC-7201 namespaced storage for the Vault module.

```solidity
struct VaultStorage {
    EnumerableSet.AddressSet assets;
    mapping(address => uint256) totalSupplies;
    mapping(address => uint256) totalBorrows;
    mapping(address => uint256) utilizationIndex;
    mapping(address => uint256) lastUpdate;
    mapping(address => bool) paused;
    address insuranceFund;
}
```

| Field             | Type                          | Description                                                        |
|-------------------|-------------------------------|--------------------------------------------------------------------|
| `assets`          | `EnumerableSet.AddressSet`    | Set of whitelisted backing assets                                  |
| `totalSupplies`   | `mapping(address => uint256)` | Total asset deposited (principal, not accounting for yield)        |
| `totalBorrows`    | `mapping(address => uint256)` | Total asset currently borrowed by Operators                        |
| `utilizationIndex`| `mapping(address => uint256)` | Cumulative time-weighted utilization integral per asset            |
| `lastUpdate`      | `mapping(address => uint256)` | Timestamp of the last index update per asset                       |
| `paused`          | `mapping(address => bool)`    | Per-asset pause state                                              |
| `insuranceFund`   | `address`                     | Address that receives minting/burning fees in cUSD                 |

---

#### `MintBurnParams`

```solidity
struct MintBurnParams {
    address asset;
    uint256 amountIn;
    uint256 amountOut;
    uint256 minAmountOut;
    address receiver;
    uint256 deadline;
    uint256 fee;
}
```

#### `RedeemParams`

```solidity
struct RedeemParams {
    uint256 amountIn;
    uint256[] amountsOut;
    uint256[] minAmountsOut;
    address receiver;
    uint256 deadline;
    uint256[] fees;
}
```

---

### Events

| Event          | Key Parameters                                        | Emitted when                          |
|----------------|-------------------------------------------------------|---------------------------------------|
| `Mint`         | `minter, receiver, asset, amountIn, amountOut, fee`   | cUSD minted                           |
| `Burn`         | `burner, receiver, asset, amountIn, amountOut, fee`   | cUSD burned for a single asset        |
| `Redeem`       | `redeemer, receiver, amountIn, amountsOut, fees`      | cUSD redeemed across all assets       |
| `Borrow`       | `borrower, asset, amount`                             | Operator borrow executed              |
| `Repay`        | `repayer, asset, amount`                              | Operator repayment received           |
| `AddAsset`     | `asset`                                               | New backing asset added               |
| `RemoveAsset`  | `asset`                                               | Backing asset removed                 |
| `PauseAsset`   | `asset`                                               | Asset paused                          |
| `UnpauseAsset` | `asset`                                               | Asset unpaused                        |

---

### Errors

| Error                  | Condition                                                    |
|------------------------|--------------------------------------------------------------|
| `PastDeadline`         | `block.timestamp > deadline`                                 |
| `Slippage`             | `amountOut < minAmountOut`                                   |
| `InvalidAmount`        | `amountOut == 0`                                             |
| `AssetPaused`          | Mint/burn/borrow attempted on a paused asset                 |
| `AssetNotSupported`    | Asset is not in the vault's asset list                       |
| `AssetAlreadySupported`| `addAsset` called for an already-listed asset                |
| `AssetHasSupplies`     | `removeAsset` called while `totalSupplies > 0`               |
| `AssetNotRescuable`    | `rescueERC20` called for a listed or strategy-held asset     |
| `InvalidMinAmountsOut` | `_minAmountsOut.length` doesn't match `assets().length`      |
| `InsufficientReserves` | Available balance is less than the amount requested          |

---

## Usage Examples

### 1. Minting cUSD

```solidity
interface IVault {
    function mint(address _asset, uint256 _amountIn, uint256 _minAmountOut, address _receiver, uint256 _deadline)
        external returns (uint256 amountOut);
    function getMintAmount(address _user, address _asset, uint256 _amountIn)
        external view returns (uint256 amountOut, uint256 fee);
}

address vault    = 0x...; // cUSD vault proxy
address usdc     = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
uint256 amountIn = 10_000e6; // 10,000 USDC

// 1. Preview the mint
(uint256 expectedOut, ) = IVault(vault).getMintAmount(msg.sender, usdc, amountIn);

// 2. Approve and mint
IERC20(usdc).approve(vault, amountIn);
uint256 cusdReceived = IVault(vault).mint(
    usdc,
    amountIn,
    expectedOut * 99 / 100, // 1% slippage tolerance
    msg.sender,
    block.timestamp + 5 minutes
);
```

### 2. Checking vault utilization before borrowing

```solidity
interface IVault {
    function availableBalance(address _asset) external view returns (uint256);
    function utilization(address _asset) external view returns (uint256 ratio);
    function totalSupplies(address _asset) external view returns (uint256);
    function totalBorrows(address _asset) external view returns (uint256);
}

function vaultStatus(address vault, address asset)
    external
    view
    returns (uint256 available, uint256 utilizationPct)
{
    available       = IVault(vault).availableBalance(asset);
    utilizationPct  = IVault(vault).utilization(asset) / 1e25; // convert ray to bps (1e27 → 100%)
}
```

### 3. Redeeming cUSD across all assets

```solidity
interface IVault {
    function assets() external view returns (address[] memory);
    function getRedeemAmount(address _user, uint256 _amountIn)
        external view returns (uint256[] memory amountsOut, uint256[] memory fees);
    function redeem(uint256 _amountIn, uint256[] calldata _minAmountsOut, address _receiver, uint256 _deadline)
        external returns (uint256[] memory amountsOut);
}

address vault      = 0x...;
uint256 cusdAmount = 5_000e18;

// Preview
(uint256[] memory expected, ) = IVault(vault).getRedeemAmount(msg.sender, cusdAmount);

// Apply 1% slippage tolerance to each asset
uint256[] memory minOut = new uint256[](expected.length);
for (uint256 i; i < expected.length; i++) {
    minOut[i] = expected[i] * 99 / 100;
}

IVault(vault).redeem(cusdAmount, minOut, msg.sender, block.timestamp + 5 minutes);
```
