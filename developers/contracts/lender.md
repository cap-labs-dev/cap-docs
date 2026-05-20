# Lender

The Lender is the lending pool that allows whitelisted Operators to borrow backing assets from the cUSD vault against delegated collateral. It tracks debt per Operator per asset, accrues two separate interest streams (vault interest paid to stcUSD holders, restaker interest paid to Delegators), and manages a liquidation system with a time-windowed grace period and linearly increasing bonus. All health factors, LTV ratios, and rates are in ray format where `1e27 = 100%`.

## Mechanics

### Borrow Flow

1. **Realize restaker interest**: Any accrued restaker interest for the Operator is realized first, updating their debt balance before new principal is added.
2. **Max borrow calculation** (if `maxBorrow = true`): `ViewLogic.maxBorrowable` computes remaining capacity based on the Operator's delegation coverage and current LTV.
3. **Validation**: `ValidationLogic.validateBorrow` checks that the Operator is whitelisted, not paused, the asset is not paused, minimum borrow is met, and vault liquidity is sufficient.
4. **State updates**: `IDelegation.setLastBorrow` is called; the Operator is marked as borrowing the reserve.
5. **Asset transfer**: The vault borrows the asset and sends it to `_receiver`; a debt token is minted to the Operator; `reserve.debt` is incremented.

### Repay Flow

1. **Realize restaker interest**: Accrued restaker interest is settled first.
2. **Amount cap**: Repayment is silently capped at the Operator's total debt. If the remaining debt after repayment would fall below `minBorrow`, the repayment is reduced to leave exactly `minBorrow` outstanding.
3. **Repayment allocation** (in priority order):
   - **Vault interest** (`interestRepaid`): Any excess above `reserve.debt + unrealizedInterest` goes to `interestReceiver`.
   - **Restaker interest** (`restakerRepaid`): Settled unrealized restaker interest is sent to the Delegation contract and distributed to Delegators.
   - **Vault principal** (`vaultRepaid`): Repays `reserve.debt` and returns assets to the vault.
4. **Debt token burn**: The debt token is burned for the full `repaid` amount.

### Interest Accrual

Cap uses two separate interest streams, both accrued continuously but realized discretely:

**Vault interest** — paid to stcUSD holders via the FeeAuction:
- Accrual is tracked by the RateOracle's utilization index (time-weighted borrow rate).
- Realized by calling `realizeInterest(_asset)` — permissionless.
- The Lender borrows the interest amount from the vault and sends it directly to `interestReceiver`.

**Restaker interest** — paid to Delegators:
- Accrues per-Operator per-asset: `debtBalance × restakerRate × elapsedTime / SECONDS_IN_YEAR`.
- `restakerRate` is set per-Operator by admin via the oracle.
- Realized on every borrow and repay, or permissionlessly via `realizeRestakerInterest`.
- If vault liquidity is insufficient, the shortfall becomes **unrealized interest** — added to the Operator's debt balance and settled during future repayments.

### Liquidation Flow

Liquidation is a two-step process:

**Step 1 — Open liquidation window:**
1. Anyone calls `openLiquidation(_agent)` when `health < 1e27` (below 100%).
2. `liquidationStart[agent]` is recorded. Reverts if a non-expired window is already open.

**Step 2 — Liquidate:**
1. `liquidate` is callable after `grace` seconds have elapsed since `liquidationStart`.
2. The liquidator repays up to `maxLiquidatable` of the Operator's debt for one asset.
3. A **bonus** is awarded in collateral value: starts at 0 after grace, grows linearly to `bonusCap` by the time `expiry` is reached. Emergency liquidations (health dangerously low) receive the full `bonusCap` immediately.
4. `IDelegation.slash` is called to seize the bonus value from the Operator's Delegators.
5. If health recovers to `≥ 1e27` after the liquidation, the window is automatically closed.

**Emergency liquidations**: If `totalDelegation × emergencyLiquidationThreshold / totalDebt < 1e27`, the grace period is skipped and the full bonus is applied immediately.

### Health Factor

```
health = (totalDelegation × liquidationThreshold) / totalDebt
```

- `health ≥ 1e27` (≥ 100%): Healthy — borrowing allowed, no liquidation.
- `health < 1e27` (< 100%): Undercollateralised — liquidation window can be opened.
- `totalDebt = 0`: Health returns `type(uint256).max`.

All values in USD with 8 decimals. `liquidationThreshold` and `ltv` are set per-Operator by the Delegation contract.

---

## Architecture

### Core Functions

#### `borrow(address _asset, uint256 _amount, address _receiver)`

```solidity
function borrow(address _asset, uint256 _amount, address _receiver) external returns (uint256 borrowed);
```

Called by a whitelisted Operator. `msg.sender` is the Operator. Pass `_amount = 0` with `maxBorrow = true` (via the BorrowParams struct internally) to borrow the maximum available.

| Parameter   | Type      | Description                            |
|-------------|-----------|----------------------------------------|
| `_asset`    | `address` | Asset to borrow                        |
| `_amount`   | `uint256` | Amount to borrow in asset decimals     |
| `_receiver` | `address` | Address to receive the borrowed assets |

**Returns**: `borrowed` — actual amount borrowed (may be less if vault liquidity is limited).

---

#### `repay(address _asset, uint256 _amount, address _agent)`

```solidity
function repay(address _asset, uint256 _amount, address _agent) external returns (uint256 repaid);
```

Repay debt on behalf of `_agent`. `msg.sender` provides the assets via `transferFrom`. Pass `_amount = type(uint256).max` to repay the full balance.

| Parameter | Type      | Description                              |
|-----------|-----------|------------------------------------------|
| `_asset`  | `address` | Asset to repay                           |
| `_amount` | `uint256` | Amount to repay in asset decimals        |
| `_agent`  | `address` | Operator whose debt is being repaid      |

**Returns**: `repaid` — actual amount repaid.

---

#### `realizeInterest(address _asset)`

```solidity
function realizeInterest(address _asset) external returns (uint256 actualRealized);
```

Permissionless. Borrows accrued vault interest from the vault and sends it to `interestReceiver` (typically the FeeAuction). Reverts with `ZeroRealization` if there is nothing to realize.

---

#### `realizeRestakerInterest(address _agent, address _asset)`

```solidity
function realizeRestakerInterest(address _agent, address _asset) external returns (uint256 actualRealized);
```

Permissionless. Realizes the accrued restaker interest for `_agent` for `_asset`. Borrows from the vault and sends to the Delegation contract for distribution to Delegators. If vault liquidity is insufficient, the shortfall is recorded as unrealized interest.

---

#### `openLiquidation(address _agent)`

```solidity
function openLiquidation(address _agent) external;
```

Opens the liquidation window for `_agent`. Requires `health < 1e27`. Reverts if an active, non-expired window already exists.

---

#### `closeLiquidation(address _agent)`

```solidity
function closeLiquidation(address _agent) external;
```

Closes the liquidation window when `health ≥ 1e27`. Anyone can call this once the Operator is healthy again.

---

#### `liquidate(address _agent, address _asset, uint256 _amount, uint256 _minLiquidatedValue)`

```solidity
function liquidate(
    address _agent,
    address _asset,
    uint256 _amount,
    uint256 _minLiquidatedValue
) external returns (uint256 liquidatedValue);
```

Repays up to `_amount` of `_agent`'s debt in `_asset` and receives slashed collateral value in return. Requires an open liquidation window past the grace period.

| Parameter             | Type      | Description                                                          |
|-----------------------|-----------|----------------------------------------------------------------------|
| `_agent`              | `address` | Operator to liquidate                                                |
| `_asset`              | `address` | Asset to repay on the Operator's behalf                              |
| `_amount`             | `uint256` | Amount of `_asset` to repay (capped at `maxLiquidatable`)            |
| `_minLiquidatedValue` | `uint256` | Minimum USD value of collateral to receive (slippage protection)     |

**Returns**: `liquidatedValue` — USD value (8 decimals) of collateral slashed and awarded to the liquidator.

---

#### View Functions

##### `agent(address _agent)`

```solidity
function agent(address _agent) external view returns (
    uint256 totalDelegation,
    uint256 totalSlashableCollateral,
    uint256 totalDebt,
    uint256 ltv,
    uint256 liquidationThreshold,
    uint256 health
);
```

Returns a complete snapshot of the Operator's collateral and health position. All USD values use 8 decimals; ratios are in ray (`1e27 = 100%`).

---

##### `maxBorrowable(address _agent, address _asset)`

```solidity
function maxBorrowable(address _agent, address _asset) external view returns (uint256 maxBorrowableAmount);
```

Returns the maximum additional amount of `_asset` the Operator can borrow given their current delegation and debt.

---

##### `maxLiquidatable(address _agent, address _asset)`

```solidity
function maxLiquidatable(address _agent, address _asset) external view returns (uint256 maxLiquidatableAmount);
```

Returns the maximum amount of `_asset` that can be liquidated against `_agent` in a single call to reach `targetHealth`.

---

##### `bonus(address _agent)`

```solidity
function bonus(address _agent) external view returns (uint256 maxBonus);
```

Returns the current liquidation bonus in ray format. Grows linearly from 0 (at grace) to `bonusCap` (at expiry). Returns `bonusCap` immediately for emergency liquidations.

---

##### `debt(address _agent, address _asset)`

```solidity
function debt(address _agent, address _asset) external view returns (uint256 totalDebt);
```

Returns the Operator's total debt for `_asset` including accrued restaker interest not yet settled.

---

##### `accruedRestakerInterest(address _agent, address _asset)`

```solidity
function accruedRestakerInterest(address _agent, address _asset) external view returns (uint256 accruedInterest);
```

Returns the restaker interest accrued since the last realization: `debt × rate × elapsed / SECONDS_IN_YEAR`.

---

##### `maxRealization(address _asset)` / `maxRestakerRealization(address _agent, address _asset)`

```solidity
function maxRealization(address _asset) external view returns (uint256 _maxRealization);
function maxRestakerRealization(address _agent, address _asset)
    external view returns (uint256 newRealizedInterest, uint256 newUnrealizedInterest);
```

Preview functions for how much interest can be realized right now vs. how much would become unrealized due to vault liquidity constraints.

---

#### Admin Functions

| Function | Description |
|----------|-------------|
| `addAsset(AddAssetParams)` | Add a new reserve (asset, vault, debt token, interest receiver) |
| `removeAsset(address)` | Remove a reserve with zero outstanding debt |
| `pauseAsset(address, bool)` | Pause or unpause borrowing for an asset |
| `setInterestReceiver(address, address)` | Update the vault interest recipient for an asset |
| `setMinBorrow(address, uint256)` | Set minimum borrow floor for an asset |
| `setGrace(uint256)` | Update the liquidation grace period (seconds) |
| `setExpiry(uint256)` | Update the liquidation expiry window (seconds) |
| `setBonusCap(uint256)` | Update the maximum liquidation bonus (ray) |

---

### Data Structures

#### `LenderStorage`

```solidity
struct LenderStorage {
    address delegation;
    address oracle;
    mapping(address => ReserveData) reservesData;
    mapping(uint256 => address) reservesList;
    uint16 reservesCount;
    mapping(address => AgentConfigurationMap) agentConfig;
    mapping(address => uint256) liquidationStart;
    uint256 targetHealth;
    uint256 grace;
    uint256 expiry;
    uint256 bonusCap;
    uint256 emergencyLiquidationThreshold;
}
```

| Field                        | Type      | Description                                                              |
|------------------------------|-----------|--------------------------------------------------------------------------|
| `delegation`                 | `address` | Delegation contract — provides coverage, LTV, and slash functions        |
| `oracle`                     | `address` | Oracle — provides asset prices and per-Operator restaker rates           |
| `reservesData`               | `mapping` | Reserve configuration and state per asset                                |
| `targetHealth`               | `uint256` | Health ratio to restore after liquidation (ray, `1e27 = 100%`)           |
| `grace`                      | `uint256` | Seconds after `openLiquidation` before liquidation is allowed            |
| `expiry`                     | `uint256` | Seconds after which a liquidation window expires if unused               |
| `bonusCap`                   | `uint256` | Maximum liquidation bonus (ray)                                          |
| `emergencyLiquidationThreshold` | `uint256` | Ratio below which emergency liquidation rules apply (ray)             |

#### `ReserveData`

```solidity
struct ReserveData {
    uint256 id;
    address vault;
    address debtToken;
    address interestReceiver;
    uint8 decimals;
    bool paused;
    uint256 debt;
    uint256 totalUnrealizedInterest;
    mapping(address => uint256) unrealizedInterest;
    mapping(address => uint256) lastRealizationTime;
    uint256 minBorrow;
}
```

| Field                    | Description                                                             |
|--------------------------|-------------------------------------------------------------------------|
| `vault`                  | cUSD vault that holds the backing assets for this reserve               |
| `debtToken`              | Non-transferrable ERC20 tracking each Operator's outstanding principal  |
| `interestReceiver`       | Receives vault interest (typically the FeeAuction)                      |
| `debt`                   | Total vault principal currently borrowed across all Operators           |
| `totalUnrealizedInterest`| Sum of all per-Operator unrealized restaker interest                    |
| `unrealizedInterest`     | Per-Operator restaker interest that couldn't be realized due to illiquidity |
| `lastRealizationTime`    | Timestamp of the last restaker interest realization per Operator        |
| `minBorrow`              | Minimum outstanding balance — partial repayments that would leave less than this are rejected |

---

## Usage Examples

### 1. Operator borrows an asset

```solidity
interface ILender {
    function borrow(address _asset, uint256 _amount, address _receiver) external returns (uint256 borrowed);
    function maxBorrowable(address _agent, address _asset) external view returns (uint256);
    function agent(address _agent) external view returns (
        uint256, uint256, uint256, uint256, uint256, uint256 health
    );
}

address lender = 0x...;
address usdc   = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

// Check health before borrowing
(,,,,,uint256 health) = ILender(lender).agent(msg.sender);
require(health >= 1e27, "Undercollateralised");

// Check capacity
uint256 capacity = ILender(lender).maxBorrowable(msg.sender, usdc);

// Borrow
uint256 borrowed = ILender(lender).borrow(usdc, capacity, msg.sender);
```

### 2. Keeper realizes vault interest

```solidity
interface ILender {
    function maxRealization(address _asset) external view returns (uint256);
    function realizeInterest(address _asset) external returns (uint256 actualRealized);
}

address lender = 0x...;
address usdc   = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

uint256 available = ILender(lender).maxRealization(usdc);
if (available > 0) {
    ILender(lender).realizeInterest(usdc);
    // Interest is now in the FeeAuction, available for stcUSD holders
}
```

### 3. Liquidator opens and executes a liquidation

```solidity
interface ILender {
    function agent(address) external view returns (uint256, uint256, uint256, uint256, uint256, uint256);
    function openLiquidation(address _agent) external;
    function maxLiquidatable(address _agent, address _asset) external view returns (uint256);
    function bonus(address _agent) external view returns (uint256);
    function liquidate(address _agent, address _asset, uint256 _amount, uint256 _minValue)
        external returns (uint256 liquidatedValue);
    function grace() external view returns (uint256);
    function liquidationStart(address) external view returns (uint256);
}

address lender = 0x...;
address agent  = 0x...; // Undercollateralised Operator
address usdc   = 0x...;

// Step 1: Open the liquidation window
(,,,,,uint256 health) = ILender(lender).agent(agent);
require(health < 1e27, "Operator is healthy");
ILender(lender).openLiquidation(agent);

// Step 2: Wait for grace period, then liquidate
// (off-chain: poll until block.timestamp >= liquidationStart + grace)
uint256 maxAmount = ILender(lender).maxLiquidatable(agent, usdc);
IERC20(usdc).approve(lender, maxAmount);

uint256 collateralValue = ILender(lender).liquidate(agent, usdc, maxAmount, 0);
// collateralValue in USD (8 decimals) has been slashed from the Operator's Delegators
```
