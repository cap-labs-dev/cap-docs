# Fractional Reserve

The Fractional Reserve layer sits inside the cUSD vault and puts idle backing assets to work in external ERC4626 yield vaults (e.g. Yearn, Aave, TokenizedStrategy vaults) while keeping a configurable liquid reserve on hand to service borrows and withdrawals without delay. When the vault needs liquidity — for a burn, redeem, or Operator borrow — it automatically divests from the yield vault before transferring funds. Any yield above principal is forwarded to the `interestReceiver` (the FeeAuction).

## Mechanics

### Invest Flow

Idle capital above the configured reserve floor is deposited into the ERC4626 yield vault:

1. **Balance check**: The vault's current `balanceOf(asset)` is compared against `reserve[asset]`.
2. **Invest**: If balance exceeds the reserve, `balance - reserve` is deposited into the ERC4626 vault via `deposit()`. The invested amount is tracked in `loaned[asset]`.
3. **Access-controlled**: `investAll` requires the `checkAccess` modifier — typically called by an admin or keeper after new deposits accumulate.

### Partial Divest Flow

When a withdrawal, redemption, or borrow needs assets that aren't in the reserve:

1. **Balance check**: If `withdrawAmount > balanceOf(asset)`, a divest is triggered.
2. **Divest amount**: The logic withdraws `withdrawAmount + reserve[asset] - balanceOf(asset)` — enough to cover the request *and* refill the buffer reserve.
3. **Share cap**: If the required shares exceed what the vault holds, the full vault balance is redeemed instead.
4. **Interest pass-through**: If `divestAmount > loaned[asset]`, the excess is forwarded to `interestReceiver` and `loaned[asset]` is reduced only by principal.

### Full Divest Flow

Used when migrating to a new yield vault or during emergency divestment:

1. All ERC4626 shares held by the contract are redeemed.
2. If `redeemedAssets > loaned[asset]`: surplus forwarded to `interestReceiver`.
3. If `redeemedAssets < loaned[asset]`: reverts with `FullDivestRequired` — a loss in the strategy, requiring human intervention.

### Interest Realization

Interest accrues passively as the ERC4626 vault appreciates. It can be realized at any time — permissionlessly via `realizeInterest`:

1. `claimableInterest` is computed: `vaultShares - sharesRequiredToWithdrawLoaned`.
2. The excess shares are redeemed and sent directly to `interestReceiver`.

The Harvester contract automates this flow: it calls `report()` on each strategy in the vault's queue, processes each report via `process_report`, then calls `realizeInterest`.

### Reserve Buffer

`reserve[asset]` is a minimum floor of liquid assets the vault always keeps on hand. Setting it higher improves withdrawal UX (fewer divestments) at the cost of yield. Setting it to 0 means all idle capital is deployed.

### LimitModule

The `LimitModule` is a companion contract deployed alongside each ERC4626 vault. It enforces that only the cUSD vault can deposit or withdraw from the fractional reserve vault, preventing external actors from interfering with the strategy.

### TokenHolder

`TokenHolder` is a minimal ERC4626-compatible strategy (built on Yearn's `BaseStrategy`) that simply holds tokens without deploying them. It is used when a strategy slot is needed but no external yield is desired — for example, holding a stablecoin in reserve during an audit or migration period.

---

## Architecture

### Core Functions

#### `investAll(address _asset)`

```solidity
function investAll(address _asset) external;
```

Access-controlled. Deposits all idle balance above `reserve[asset]` into the configured ERC4626 vault for `_asset`. No-ops if no vault is set or if balance ≤ reserve.

---

#### `divestAll(address _asset)`

```solidity
function divestAll(address _asset) external;
```

Access-controlled. Fully redeems all shares from the ERC4626 vault for `_asset`. Surplus above `loaned[asset]` is forwarded to `interestReceiver`. Reverts with `FullDivestRequired` if a loss has occurred.

---

#### `setFractionalReserveVault(address _asset, address _vault)`

```solidity
function setFractionalReserveVault(address _asset, address _vault) external;
```

Access-controlled. Migrates `_asset` to a new ERC4626 vault. The existing vault is fully divested first. Reverts if `_vault` is already registered for another asset.

---

#### `setReserve(address _asset, uint256 _reserve)`

```solidity
function setReserve(address _asset, uint256 _reserve) external;
```

Access-controlled. Sets the minimum liquid buffer for `_asset` in asset-native units. Funds above this threshold are eligible for investment.

---

#### `realizeInterest(address _asset)`

```solidity
function realizeInterest(address _asset) external;
```

Permissionless. Withdraws accrued yield (ERC4626 appreciation above `loaned[asset]`) and sends it to `interestReceiver`. No-ops if no yield has accrued or no vault is set.

---

#### `claimableInterest(address _asset)`

```solidity
function claimableInterest(address _asset) external view returns (uint256 interest);
```

Returns the amount of `_asset` that can currently be claimed as yield. Computed as the asset value of the shares held in excess of those needed to recover `loaned[asset]`.

---

#### View Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `fractionalReserveVault(address _asset)` | `address` | ERC4626 vault address configured for `_asset` |
| `fractionalReserveVaults()` | `address[]` | All registered ERC4626 vault addresses |
| `reserve(address _asset)` | `uint256` | Configured liquid reserve floor for `_asset` |
| `loaned(address _asset)` | `uint256` | Principal currently deposited in the yield vault for `_asset` |
| `interestReceiver()` | `address` | Address that receives realized yield (FeeAuction) |

---

### Harvester

The `Harvester` is a stateless keeper contract. A keeper (e.g. Gelato) calls `harvest` periodically to report strategy performance and realize interest.

#### `harvest(address _cusd, address _asset)`

```solidity
function harvest(address _cusd, address _asset)
    external
    returns (uint256 profit, uint256 loss, uint256 interest);
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `_cusd`   | `address` | The cUSD vault proxy address |
| `_asset`  | `address` | The backing asset to harvest for |

**Returns**:
- `profit` — cumulative profit across all strategies in the vault queue
- `loss` — cumulative loss across all strategies
- `interest` — amount of interest realized and sent to the fee auction

**Process flow**:
1. Fetches the ERC4626 vault's `default_queue` (ordered list of strategies).
2. For each strategy: calls `strategy.report()` then `vault.process_report(strategy)`.
3. Calls `realizeInterest(_asset)` if any claimable interest exists.

---

### LimitModule

A deployment-time immutable contract that gates deposits and withdrawals in the ERC4626 vault to the cUSD vault only.

#### `available_deposit_limit(address receiver)`

```solidity
function available_deposit_limit(address receiver) external view returns (uint256 limit);
```

Returns `type(uint256).max` if `receiver == vault`, otherwise 0.

#### `available_withdraw_limit(address owner, uint256, address[] calldata)`

```solidity
function available_withdraw_limit(address owner, uint256, address[] calldata)
    external view returns (uint256 limit);
```

Returns `type(uint256).max` if `owner == vault`, otherwise 0.

---

### TokenHolder

A minimal Yearn `BaseStrategy` that holds tokens in place without deploying them to an external yield source. `_deployFunds` and `_freeFunds` are intentionally empty. Deposit and withdrawal are restricted to `vault`.

---

### Data Structures

#### `FractionalReserveStorage`

```solidity
struct FractionalReserveStorage {
    address interestReceiver;
    mapping(address => uint256) loaned;
    mapping(address => uint256) reserve;
    mapping(address => address) vault;
    EnumerableSet.AddressSet vaults;
}
```

| Field              | Type                          | Description                                                        |
|--------------------|-------------------------------|--------------------------------------------------------------------|
| `interestReceiver` | `address`                     | Receives yield forwarded from the fractional reserve (FeeAuction)  |
| `loaned`           | `mapping(address => uint256)` | Principal deposited into each asset's yield vault                  |
| `reserve`          | `mapping(address => uint256)` | Minimum liquid buffer to maintain per asset (in asset units)       |
| `vault`            | `mapping(address => address)` | ERC4626 yield vault address per backing asset                      |
| `vaults`           | `EnumerableSet.AddressSet`    | Set of all registered yield vault addresses (for uniqueness checks)|

---

### Events

| Event | Parameters | Emitted when |
|-------|-----------|--------------|
| `FractionalReserveInvested` | `asset, amount` | Assets deposited into the yield vault |
| `FractionalReserveDivested` | `asset, amount` | Assets withdrawn from the yield vault |
| `FractionalReserveVaultUpdated` | `asset, vault` | Yield vault configured or changed for an asset |
| `FractionalReserveReserveUpdated` | `asset, reserve` | Reserve buffer updated for an asset |
| `FractionalReserveInterestRealized` | `asset, amount` | Yield forwarded to `interestReceiver` |

---

### Errors

| Error | Condition |
|-------|-----------|
| `FullDivestRequired` | Full divest resulted in fewer assets than `loaned` — strategy loss |
| `FractionalReserveVaultAlreadySet` | `_vault` is already registered for a different asset |

---

## Usage Examples

### 1. Keeper harvest job

```solidity
interface IHarvester {
    function harvest(address _cusd, address _asset)
        external
        returns (uint256 profit, uint256 loss, uint256 interest);
}

address harvester = 0x...;
address cusd      = 0x...; // cUSD vault proxy
address usdc      = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

// Called by a Gelato task or cron keeper
(uint256 profit, uint256 loss, uint256 interest) = IHarvester(harvester).harvest(cusd, usdc);
// interest is now in the FeeAuction, available for stcUSD holders to bid on
```

### 2. Checking available liquidity before a large redemption

```solidity
interface IFractionalReserve {
    function loaned(address _asset) external view returns (uint256);
    function claimableInterest(address _asset) external view returns (uint256);
}

interface IVault {
    function availableBalance(address _asset) external view returns (uint256);
}

function checkLiquidity(address vault, address asset)
    external
    view
    returns (uint256 liquid, uint256 investedPrincipal, uint256 pendingYield)
{
    liquid            = IVault(vault).availableBalance(asset);   // on-hand balance
    investedPrincipal = IFractionalReserve(vault).loaned(asset); // in yield vault
    pendingYield      = IFractionalReserve(vault).claimableInterest(asset);
}
```

### 3. Admin: onboarding a new yield vault for USDC

```solidity
interface IFractionalReserve {
    function setFractionalReserveVault(address _asset, address _vault) external;
    function setReserve(address _asset, uint256 _reserve) external;
    function investAll(address _asset) external;
}

address vault = 0x...; // cUSD vault proxy
address usdc  = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
address yvUSDC = 0x...; // New ERC4626 vault

// 1. Set the new yield vault (automatically divests old one if set)
IFractionalReserve(vault).setFractionalReserveVault(usdc, yvUSDC);

// 2. Set a 500,000 USDC liquid reserve floor
IFractionalReserve(vault).setReserve(usdc, 500_000e6);

// 3. Deploy idle balance into the new vault
IFractionalReserve(vault).investAll(usdc);
```
