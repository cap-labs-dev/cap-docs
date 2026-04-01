# Tokens

Cap has three token contracts: **cUSD** (the backing stablecoin vault), **stcUSD** (the yield-bearing staked version), and **L2Token** (a LayerZero OFT for cross-chain bridging). cUSD and stcUSD are deployed on mainnet; L2Token is deployed on destination chains.

## cUSD — CapToken

`CapToken` is the cUSD stablecoin. It is a UUPS upgradeable contract that inherits the full `Vault`, `Minter`, and `FractionalReserve` modules — see those pages for the complete function reference. The token itself is an ERC20 whose supply is backed 1:1 (plus protocol fees) by a basket of whitelisted backing assets.

**Key properties**:
- Minted by depositing backing assets (`mint`)
- Burned to withdraw backing assets (`burn`) or proportionally across all assets (`redeem`)
- Supply is managed by the Vault module — no direct `mint`/`burn` by admin
- Price is the weighted average of underlying asset values (via `CapTokenAdapter`)

### `initialize`

```solidity
function initialize(
    string memory _name,
    string memory _symbol,
    address _accessControl,
    address _feeAuction,
    address _oracle,
    address[] calldata _assets,
    address _insuranceFund
) external initializer;
```

| Parameter | Description |
|-----------|-------------|
| `_name` / `_symbol` | ERC20 name and symbol (e.g. "Cap USD", "cUSD") |
| `_accessControl` | AccessControl contract address |
| `_feeAuction` | FeeAuction address — receives fractional reserve yield |
| `_oracle` | Oracle contract address |
| `_assets` | Initial list of whitelisted backing assets |
| `_insuranceFund` | Receives mint/burn fees in cUSD |

---

## stcUSD — StakedCap

`StakedCap` is a UUPS upgradeable ERC4626 vault that accepts cUSD deposits and distributes protocol yield to holders. Yield accrues as the Lender realizes vault interest and transfers cUSD to this contract. A **profit-locking** mechanism prevents flashloan manipulation by vesting newly notified yield linearly over `lockDuration` seconds.

### Mechanics

#### Deposit and Withdrawal

StakedCap is a standard ERC4626: deposit cUSD, receive stcUSD shares proportional to the current exchange rate. Withdraw shares to receive cUSD plus accrued yield.

```
sharePrice = totalAssets() / totalSupply()
totalAssets() = storedTotal - lockedProfit()
```

`storedTotal` is the raw cUSD balance held by the contract. `lockedProfit()` is the portion of the most recent yield notification still vesting — it decays linearly to zero over `lockDuration`.

#### `notify()`

```solidity
function notify() external;
```

Permissionless. Snapshots the current balance above `storedTotal` as new locked profit and starts the vesting timer. Reverts with `StillVesting` if the previous yield batch has not fully unlocked yet — this prevents yield dilution by stacking notifications.

**Typical flow**: The FeeAuction's `paymentRecipient` is the StakedCap contract. After a buyer purchases auction assets, cUSD flows to StakedCap. Any keeper (or the buyer themselves) then calls `notify()` to begin vesting.

#### `lockedProfit()`

```solidity
function lockedProfit() external view returns (uint256 locked);
```

Returns the portion of the last notified yield still locked. Decays linearly from `totalLocked` at `lastNotify` to zero at `lastNotify + lockDuration`.

```
lockedProfit = totalLocked × (lockDuration - elapsed) / lockDuration
```

Returns zero if `elapsed ≥ lockDuration`.

---

### View Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `totalAssets()` | `uint256` | `storedTotal - lockedProfit()` — effective cUSD backing shares |
| `lockedProfit()` | `uint256` | Yield still vesting (decays to 0 over `lockDuration`) |
| `lastNotify()` | `uint256` | Timestamp of the most recent `notify()` call |
| `lockDuration()` | `uint256` | Seconds over which newly notified yield vests |
| `totalLocked()` | `uint256` | Total cUSD locked at the time of the last notify |

---

### `initialize`

```solidity
function initialize(
    address _accessControl,
    address _asset,
    uint256 _lockDuration
) external initializer;
```

| Parameter | Description |
|-----------|-------------|
| `_accessControl` | AccessControl contract |
| `_asset` | cUSD token address (the underlying ERC4626 asset) |
| `_lockDuration` | Yield vesting period in seconds |

The contract's ERC20 name and symbol are derived automatically from the underlying cUSD token: `"Staked " + name` and `"st" + symbol`.

---

## L2Token

`L2Token` is a LayerZero OFT (Omnichain Fungible Token) with EIP-2612 permit support. It is deployed on L2/alt-chain destinations to represent bridged cUSD or other Cap tokens.

**Key properties**:
- Standard `OFT` from the LayerZero SDK — `send` and `receive` cross-chain transfers
- `permit` support via EIP-2612 for gasless approvals
- Not upgradeable — deployed with a fixed constructor
- Cross-chain supply is managed by the LayerZero endpoint; no manual minting

### Constructor

```solidity
constructor(
    string memory _name,
    string memory _symbol,
    address _lzEndpoint,
    address _delegate
)
```

| Parameter | Description |
|-----------|-------------|
| `_name` / `_symbol` | ERC20 name and symbol |
| `_lzEndpoint` | LayerZero endpoint contract on this chain |
| `_delegate` | Address authorized to configure the OApp (peer chains, gas limits, etc.) |

---

## Usage Examples

### 1. Staking cUSD to earn yield

```solidity
interface IStakedCap {
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);
    function totalAssets() external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
}

address stcusd  = 0x...;
address cusd    = 0x...;
uint256 deposit = 10_000e18; // 10,000 cUSD

// Approve and deposit
IERC20(cusd).approve(stcusd, deposit);
uint256 shares = IStakedCap(stcusd).deposit(deposit, msg.sender);

// Later: check accrued value
uint256 currentValue = IStakedCap(stcusd).convertToAssets(shares);
// currentValue > deposit once yield has been notified and vested
```

---

### 2. Notifying new yield

```solidity
interface IStakedCap {
    function notify() external;
    function lockedProfit() external view returns (uint256);
    function lastNotify() external view returns (uint256);
    function lockDuration() external view returns (uint256);
}

address stcusd = 0x...;

// Check if previous yield has finished vesting before notifying
uint256 locked = IStakedCap(stcusd).lockedProfit();
if (locked == 0) {
    IStakedCap(stcusd).notify(); // starts new vesting cycle
}
```

---

### 3. Checking the cUSD vault composition

```solidity
interface ICapToken {
    function assets() external view returns (address[] memory);
    function totalSupplies(address _asset) external view returns (uint256);
    function totalBorrows(address _asset) external view returns (uint256);
    function availableBalance(address _asset) external view returns (uint256);
    function utilization(address _asset) external view returns (uint256 ratio);
}

address cusd = 0x...;
address[] memory vaultAssets = ICapToken(cusd).assets();

for (uint i; i < vaultAssets.length; i++) {
    address asset = vaultAssets[i];
    uint256 supply    = ICapToken(cusd).totalSupplies(asset);
    uint256 borrowed  = ICapToken(cusd).totalBorrows(asset);
    uint256 available = ICapToken(cusd).availableBalance(asset);
    uint256 util      = ICapToken(cusd).utilization(asset); // ray
    // util / 1e25 = utilization in basis points
}
```
