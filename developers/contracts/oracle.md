# Oracle

The Oracle system provides two types of data to the Cap protocol: **asset prices** (USD values for backing assets, collateral tokens, and Cap-native tokens) and **interest rates** (borrow rates for vault utilization and per-Operator restaker rates). Both systems share the same adapter architecture — a central contract stores a payload per asset, and delegates the actual calculation to a stateless library via `staticcall`. This makes the oracle fully composable: any data source can be integrated by deploying a new adapter library.

## Mechanics

### Adapter Architecture

Each asset has an `OracleData` struct stored on-chain containing two fields:
- `adapter` — the address of the library or contract that computes the value
- `payload` — ABI-encoded calldata passed to the adapter

When a price or rate is requested, the Oracle does a low-level `staticcall` to `adapter(payload)`. Errors in the adapter are silently swallowed — a failed call returns zero rather than reverting. This means any data source failure falls through to the backup rather than halting the protocol.

### Price Oracle: Primary and Backup

Each asset has two `OracleData` slots — primary and backup. `getPrice` attempts the primary first:

1. Call `primary.adapter` with `primary.payload`.
2. If the returned price is zero **or** the timestamp is older than `staleness[asset]`, fall through.
3. Call `backup.adapter` with `backup.payload`.
4. If backup also fails or is stale, revert with `PriceError`.

All prices are normalized to **8 decimals** (USD). Adapters are responsible for decimal conversion.

### Rate Oracle: Market and Utilization

Rates drive two separate interest streams:

- **Market rate** — an external floor rate (e.g. Aave borrow rate). Used as a benchmark.
- **Utilization rate** — derived from the cUSD vault's time-weighted utilization (via VaultAdapter). This is the actual rate Operators pay on vault interest.

Both are fetched via the same adapter pattern, but rate adapters use `call` rather than `staticcall` because VaultAdapter writes state (it updates the utilization index snapshot).

Additionally, two admin-set rates are stored directly:
- `benchmarkRate[asset]` — a manual floor rate per asset (ray)
- `restakerRate[agent]` — per-Operator restaker interest rate (ray)

---

## Architecture

### PriceOracle

Abstract contract. Inherited by the central `Oracle` contract.

#### `getPrice(address _asset)`

```solidity
function getPrice(address _asset) external view returns (uint256 price, uint256 lastUpdated);
```

Returns the asset's USD price (8 decimals) and the timestamp of the last update. Tries the primary adapter first; falls back to the backup if the primary returns zero or a stale price. Reverts with `PriceError` if both fail.

---

#### Admin Functions

| Function | Description |
|----------|-------------|
| `setPriceOracleData(address _asset, OracleData)` | Set primary price adapter + payload for an asset |
| `setPriceBackupOracleData(address _asset, OracleData)` | Set backup price adapter + payload for an asset |
| `setStaleness(address _asset, uint256 _staleness)` | Set max age (seconds) for a valid price |

---

#### View Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `priceOracleData(address)` | `OracleData` | Primary adapter + payload for the asset |
| `priceBackupOracleData(address)` | `OracleData` | Backup adapter + payload for the asset |
| `staleness(address)` | `uint256` | Max acceptable age in seconds for the price |

---

### RateOracle

Abstract contract. Inherited by the central `Oracle` contract.

#### `marketRate(address _asset)`

```solidity
function marketRate(address _asset) external returns (uint256 rate);
```

Returns the current market rate for `_asset` in ray format (`1e27 = 100%` annualised). Calls the configured market adapter (e.g. AaveAdapter). Returns zero if the adapter fails.

---

#### `utilizationRate(address _asset)`

```solidity
function utilizationRate(address _asset) external returns (uint256 rate);
```

Returns the utilization-based interest rate for `_asset` in ray. Calls the configured utilization adapter (VaultAdapter). This is the rate Operators pay on borrowed vault principal.

---

#### Admin Functions

| Function | Description |
|----------|-------------|
| `setMarketOracleData(address _asset, OracleData)` | Configure the market rate adapter + payload |
| `setUtilizationOracleData(address _asset, OracleData)` | Configure the utilization rate adapter + payload |
| `setBenchmarkRate(address _asset, uint256 _rate)` | Set a manual benchmark rate floor (ray) |
| `setRestakerRate(address _agent, uint256 _rate)` | Set the restaker interest rate for an Operator (ray) |

---

#### View Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `benchmarkRate(address)` | `uint256` (ray) | Manually set rate floor for the asset |
| `restakerRate(address)` | `uint256` (ray) | Per-Operator restaker interest rate |
| `marketOracleData(address)` | `OracleData` | Market adapter + payload |
| `utilizationOracleData(address)` | `OracleData` | Utilization adapter + payload |

---

### Data Structures

#### `OracleData`

```solidity
struct OracleData {
    address adapter;
    bytes payload;
}
```

| Field | Description |
|-------|-------------|
| `adapter` | Contract or library address called via `staticcall` (prices) or `call` (rates) |
| `payload` | ABI-encoded calldata passed directly to the adapter |

---

## Price Adapters

### ChainlinkAdapter

Reads a Chainlink price feed aggregator. Normalizes the returned value to 8 decimals regardless of the feed's native decimals.

**Payload**: `abi.encodeCall(ChainlinkAdapter.price, (_source))` where `_source` is the Chainlink aggregator address.

**Returns**: `(uint256 price, uint256 lastUpdated)` — price in 8 decimals, `lastUpdated` from `latestRoundData`.

---

### ChainlinkAdapterChained

Multiplies multiple Chainlink feeds together to derive a cross-rate. For example, to price an asset quoted in ETH: feed `A/ETH` × feed `ETH/USD`. The final price is normalized to 8 decimals; `lastUpdated` is the minimum across all feeds (most conservative).

**Payload**: `abi.encodeCall(ChainlinkAdapterChained.price, (_sources))` where `_sources` is an array of aggregator addresses.

---

### wstETHAdapter

Prices wstETH by reading the stETH/USD Chainlink feed and multiplying by the stETH-to-wstETH conversion ratio from the Lido contract (`getPooledEthByShares(1 ether)`).

**Payload**: `abi.encodeCall(wstETHAdapter.price, (_source, _stETH))` where `_source` is the stETH Chainlink aggregator and `_stETH` is the stETH token address.

---

### CapTokenAdapter

Prices a cUSD vault token (e.g. cUSD itself) as the weighted average of its backing assets:

```
price = Σ(totalSupplies[asset] × assetPrice / 10^assetDecimals) / capTokenSupply × 10^capDecimals
```

Calls back into the Oracle via `msg.sender` to get each underlying asset's price. `lastUpdated` is the minimum across all underlying prices. Returns `1e8` if supply is zero.

**Payload**: `abi.encodeCall(CapTokenAdapter.price, (_asset))` where `_asset` is the Cap vault token address.

---

### StakedCapAdapter

Prices stcUSD (staked cUSD) by reading the cUSD price and multiplying by the `convertToAssets` ratio of the ERC4626 staking vault:

```
price = capTokenPrice × convertToAssets(1e18) / 1e18
```

**Payload**: `abi.encodeCall(StakedCapAdapter.price, (_asset))` where `_asset` is the stcUSD address.

---

### FixedPriceOracle

A standalone contract (not a library) that always returns `1e8` (1 USD). Used for stablecoins or test deployments.

**Payload**: `abi.encodeCall(FixedPriceOracle.price, ())` — no parameters.

---

### UniswapV3Adapter

On-chain TWAP oracle using Uniswap V3 pool observations. Supports chained routes (e.g. `TOKEN → WETH → USDC`) to derive prices for any token with sufficient liquidity.

**Payload**: `abi.encode(tokens[], pools[], twapPeriods[])` where:
- `tokens` — ordered token route (length = `pools.length + 1`)
- `pools` — Uniswap V3 pool addresses for each hop
- `twapPeriods` — TWAP window in seconds per hop

The first token in the route must already have a registered price in the Oracle (used as the base price). The adapter validates the route via `validateData` before the payload is stored.

---

## Rate Adapters

### AaveAdapter

Reads Aave's `getReserveData` to fetch the current variable borrow rate for an asset. Used as a market rate benchmark.

**Payload**: `abi.encodeCall(AaveAdapter.rate, (_aaveDataProvider, _asset))`.

---

### VaultAdapter

Computes a utilization-based interest rate using Cap's own vault utilization history. This is the primary rate source for vault interest.

The adapter tracks a per-vault per-asset **utilization snapshot** (`index`, `lastUpdate`) and computes the time-weighted average utilization since the last call. The rate is determined by a two-slope model with a dynamic multiplier:

- **Below kink**: rate = `slope0 × utilization / kink × multiplier`. Multiplier decays when utilization is low.
- **Above kink**: rate = `(slope0 + slope1 × excess) × multiplier`. Multiplier grows when utilization is high.

The multiplier is bounded by `[minMultiplier, maxMultiplier]` and adjusts at speed `rate` (ray per second).

#### Admin Functions

| Function | Description |
|----------|-------------|
| `setSlopes(address _asset, SlopeData)` | Configure the two-slope curve for an asset |
| `setLimits(uint256 _max, uint256 _min, uint256 _rate)` | Set multiplier bounds and adjustment speed |

#### `SlopeData`

```solidity
struct SlopeData {
    uint256 slope0;  // ray — base rate at kink
    uint256 slope1;  // ray — additional rate above kink per unit of excess
    uint256 kink;    // ray — utilization ratio that separates the two slopes
}
```

---

## Usage Examples

### 1. Reading an asset price

```solidity
interface IOracle {
    function getPrice(address _asset) external view returns (uint256 price, uint256 lastUpdated);
}

address oracle = 0x...;
address usdc   = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

(uint256 price, uint256 updatedAt) = IOracle(oracle).getPrice(usdc);
// price = 1_0000_0000 (1.00 USD, 8 decimals)

uint256 usdValue = amount * price / 10 ** 6; // USDC has 6 decimals
```

---

### 2. Configuring a Chainlink price feed

```solidity
interface IOracle {
    function setPriceOracleData(address _asset, OracleData calldata) external;
    function setPriceBackupOracleData(address _asset, OracleData calldata) external;
    function setStaleness(address _asset, uint256 _staleness) external;
}

struct OracleData { address adapter; bytes payload; }

address oracle          = 0x...;
address chainlinkAdapter = 0x...;
address wbtcFeed        = 0x...; // Chainlink WBTC/USD aggregator
address wbtc            = 0x...; // WBTC token

// Primary: Chainlink WBTC/USD
IOracle(oracle).setPriceOracleData(wbtc, OracleData({
    adapter: chainlinkAdapter,
    payload: abi.encodeWithSignature("price(address)", wbtcFeed)
}));

// Allow up to 1 hour staleness
IOracle(oracle).setStaleness(wbtc, 3600);
```

---

### 3. Setting a restaker rate for an Operator

```solidity
interface IOracle {
    function setRestakerRate(address _agent, uint256 _rate) external;
    function restakerRate(address _agent) external view returns (uint256);
}

address oracle   = 0x...;
address operator = 0x...;

// Set 5% annual restaker rate (ray = 1e27)
IOracle(oracle).setRestakerRate(operator, 0.05e27);

// Verify
uint256 rate = IOracle(oracle).restakerRate(operator);
// rate = 5e25 (5% in ray)
```
