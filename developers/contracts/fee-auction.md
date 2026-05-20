# Fee Auction

The FeeAuction distributes protocol yield — collected as backing assets from Operator interest payments — to stcUSD holders via a Dutch auction mechanism. stcUSD holders bid cUSD at a continuously decaying price to purchase the accumulated backing assets at a discount, creating a fixed-rate yield experience. Each successful purchase resets the auction at double the settlement price, establishing a self-calibrating price discovery cycle. The FeeReceiver acts as the upstream collector, receiving vault interest from the Lender and holding it until the FeeAuction sweeps it.

## Mechanics

### Dutch Auction Price Curve

The auction price decays linearly from `startPrice` to 10% of `startPrice` over `duration` seconds:

```
price = startPrice × (1 - 0.9 × elapsed / duration)
```

- At `t = 0`: price = `startPrice` (100%)
- At `t = duration`: price = `startPrice × 0.1` (10%)
- Price never reaches zero — it floors at 10% of the start price

All price values are in the `paymentToken` (cUSD) with 18 decimals.

### Buy Flow

1. **Price check**: `currentPrice()` is compared against caller's `_maxPrice`. Reverts with `InvalidPrice` if `currentPrice > _maxPrice`.
2. **Validation**: Asset list must be non-empty and match `_minAmounts` length; `_receiver` must be non-zero; deadline must be in the future.
3. **New auction reset**: `startTimestamp` is set to `block.timestamp`; `startPrice` is set to `max(currentPrice × 2, minStartPrice)`.
4. **Asset transfer**: Full balances of each requested `_asset` held by the FeeAuction are transferred to `_receiver`. Reverts with `InsufficientBalance` if any asset balance is below `_minAmounts[i]`.
5. **Payment**: `currentPrice` in `paymentToken` is pulled from `msg.sender` and sent to `paymentRecipient` (the stcUSD distribution contract or treasury).

### Self-Calibrating Start Price

Each successful purchase sets the next auction's `startPrice` to `2 × settlementPrice`. This means:
- If buyers wait for a deep discount, the next auction starts low.
- If buyers buy early, the next auction starts high.
- The auction converges on a market-clearing rate for protocol yield.

The `minStartPrice` floor prevents the auction from becoming trivially cheap during low-activity periods.

### Interest Flow End-to-End

```
Operator repays interest
        ↓
Lender sends vault interest to interestReceiver (FeeReceiver)
        ↓
FeeReceiver holds backing assets (USDC, wETH, etc.)
        ↓
Buyer calls buy() on FeeAuction, pays cUSD at current price
        ↓
Buyer receives backing assets; cUSD goes to paymentRecipient
        ↓
paymentRecipient distributes yield to stcUSD holders
```

---

## Architecture

### FeeAuction

#### Core Functions

##### `buy(uint256 _maxPrice, address[] _assets, uint256[] _minAmounts, address _receiver, uint256 _deadline)`

```solidity
function buy(
    uint256 _maxPrice,
    address[] calldata _assets,
    uint256[] calldata _minAmounts,
    address _receiver,
    uint256 _deadline
) external;
```

Purchase all accumulated fee assets at the current Dutch auction price.

| Parameter    | Type        | Description                                                             |
|--------------|-------------|-------------------------------------------------------------------------|
| `_maxPrice`  | `uint256`   | Maximum cUSD to pay; reverts if `currentPrice > _maxPrice`              |
| `_assets`    | `address[]` | List of backing assets to receive from the auction contract             |
| `_minAmounts`| `uint256[]` | Minimum balance of each asset to accept (slippage protection per asset) |
| `_receiver`  | `address`   | Address to receive the purchased backing assets                         |
| `_deadline`  | `uint256`   | Unix timestamp after which the tx reverts                               |

---

##### `currentPrice()`

```solidity
function currentPrice() external view returns (uint256 price);
```

Returns the current auction price in `paymentToken` units. Decays linearly from `startPrice` to `startPrice × 0.1` over `duration`. Use this before calling `buy` to set `_maxPrice`.

---

#### Admin Functions

| Function | Description |
|----------|-------------|
| `setStartPrice(uint256)` | Manually reset the auction start price (must be ≥ `minStartPrice`) |
| `setDuration(uint256)` | Update the auction duration in seconds |
| `setMinStartPrice(uint256)` | Update the minimum allowed start price floor |
| `setPaymentToken(address)` | Change the payment token (cUSD by default) |

---

#### View Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `currentPrice()` | `uint256` | Current auction price in payment token |
| `startPrice()` | `uint256` | Starting price of the current auction |
| `startTimestamp()` | `uint256` | When the current auction started |
| `duration()` | `uint256` | Auction duration in seconds |
| `minStartPrice()` | `uint256` | Floor price for future auction starts |
| `paymentToken()` | `address` | Token buyers pay with (cUSD) |
| `paymentRecipient()` | `address` | Receives the cUSD payment from buyers |

---

#### Data Structures

##### `FeeAuctionStorage`

```solidity
struct FeeAuctionStorage {
    address paymentToken;
    address paymentRecipient;
    uint256 startPrice;
    uint256 startTimestamp;
    uint256 duration;
    uint256 minStartPrice;
}
```

| Field              | Description                                                                      |
|--------------------|----------------------------------------------------------------------------------|
| `paymentToken`     | ERC20 token buyers pay with — cUSD                                               |
| `paymentRecipient` | Receives cUSD from every purchase — distributes to stcUSD holders                |
| `startPrice`       | Current auction starting price in `paymentToken` (18 decimals)                   |
| `startTimestamp`   | `block.timestamp` of the last purchase (or initialization)                       |
| `duration`         | Seconds for the price to decay from `startPrice` to `startPrice × 0.1`           |
| `minStartPrice`    | Minimum `startPrice` floor — prevents the auction going below a meaningful level |

---

#### Events

| Event | Parameters | Emitted when |
|-------|-----------|--------------|
| `Buy` | `buyer, price, assets[], balances[]` | A successful purchase is made |
| `SetDuration` | `duration` | Auction duration updated |
| `SetMinStartPrice` | `minStartPrice` | Minimum start price updated |
| `SetPaymentToken` | `paymentToken` | Payment token changed |
| `SetStartPrice` | `startPrice` | Start price manually reset |

---

#### Errors

| Error | Condition |
|-------|-----------|
| `InvalidPrice` | `currentPrice > _maxPrice` |
| `InvalidAssets` | Empty asset list or `assets.length ≠ minAmounts.length` |
| `InvalidDeadline` | `_deadline < block.timestamp` |
| `InvalidReceiver` | `_receiver == address(0)` |
| `InvalidStartPrice` | `setStartPrice` called with value below `minStartPrice` |
| `InvalidPaymentToken` | `setPaymentToken` called with zero address |
| `InsufficientBalance` | Asset balance in the contract is below `_minAmounts[i]` |
| `NoMinStartPrice` | `initialize` called with `_minStartPrice == 0` |
| `NoDuration` | `initialize` or `setDuration` called with `_duration == 0` |

---

### FeeReceiver

The FeeReceiver is a simple holding contract that sits between the Lender and the FeeAuction. The Lender sends vault interest to the FeeReceiver, which holds the assets until a buyer triggers the FeeAuction sweep.

It has no complex logic — its purpose is to batch up interest from multiple reserves before each auction, so buyers receive a meaningful bundle of assets per purchase rather than tiny per-asset drips.

---

## Usage Examples

### 1. Buying auction assets (stcUSD holder / arbitrageur)

```solidity
interface IFeeAuction {
    function currentPrice() external view returns (uint256);
    function buy(
        uint256 _maxPrice,
        address[] calldata _assets,
        uint256[] calldata _minAmounts,
        address _receiver,
        uint256 _deadline
    ) external;
}

address feeAuction = 0x...;
address cusd       = 0x...;
address usdc       = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
address weth       = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

// Check current price
uint256 price = IFeeAuction(feeAuction).currentPrice();

// Approve payment
IERC20(cusd).approve(feeAuction, price);

// Specify assets to buy and minimum amounts (0 = accept any amount)
address[] memory assets     = new address[](2);
uint256[] memory minAmounts = new uint256[](2);
assets[0] = usdc; assets[1] = weth;
minAmounts[0] = 0; minAmounts[1] = 0;

// Buy — receive all USDC and WETH held by the FeeAuction
IFeeAuction(feeAuction).buy(
    price,         // exact price (no slippage on price)
    assets,
    minAmounts,
    msg.sender,
    block.timestamp + 1 minutes
);
```

### 2. Monitoring the auction price off-chain

```solidity
interface IFeeAuction {
    function currentPrice() external view returns (uint256);
    function startPrice() external view returns (uint256);
    function startTimestamp() external view returns (uint256);
    function duration() external view returns (uint256);
}

function getAuctionState(address feeAuction) external view returns (
    uint256 price,
    uint256 discountPct,   // how far the price has decayed (0-90%)
    uint256 secondsLeft    // seconds until price reaches floor
) {
    price = IFeeAuction(feeAuction).currentPrice();
    uint256 start     = IFeeAuction(feeAuction).startPrice();
    uint256 elapsed   = block.timestamp - IFeeAuction(feeAuction).startTimestamp();
    uint256 dur       = IFeeAuction(feeAuction).duration();

    discountPct = elapsed >= dur ? 90 : (elapsed * 90 / dur);
    secondsLeft = elapsed >= dur ? 0 : dur - elapsed;
}
```

### 3. Full interest flow — from Operator repayment to stcUSD yield

```
1. Operator calls Lender.repay(usdc, amount, operator)
   → Lender sends vault interest to FeeReceiver (interestReceiver)

2. FeeReceiver now holds USDC (and other backing assets from other reserves)

3. Buyer calls FeeAuction.buy(maxPrice, [usdc, weth], [0, 0], buyer, deadline)
   → Buyer pays cUSD at currentPrice
   → Buyer receives USDC + WETH from FeeAuction
   → cUSD goes to paymentRecipient

4. paymentRecipient distributes cUSD yield to stcUSD holders
   (yield rate = cUSD paid / total stcUSD supply, annualised)
```
