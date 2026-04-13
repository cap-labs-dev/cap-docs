# Fee Auction

The [Fee Auction](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/feeAuction/FeeAuction.sol) facilitates permissionless Dutch auctions where collected protocol fees are sold to the winning bidder. The proceeds are sent to the [Fee Receiver](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/feeReceiver/FeeReceiver.sol) to convert to cUSD and distribute to stcUSD holders. The Fee Auction module provides an efficient and transparent way to distribute accumulated fees to participants while ensuring fair price discovery through time-based price decay.

## Mechanics

### Overview of Operations

1. **Interest Harvesting**: Interest Harvester realizes accumulated interest and sends it to Fee Auction
2. **Dutch Auction**: Fee Auction sells accumulated assets via a Dutch auction mechanism
3. **Fee Distribution**: Fee Receiver collects cUSD from auction sales and distributes to stcUSD token holders

<figure><img src="../.gitbook/assets/image (31).png" alt=""><figcaption><p>Flow of Funds: Fee Auction</p></figcaption></figure>

### Dutch Auction Mechanics

* Admin sets a starting price (in cUSD) and the duration of auctions.
* The price decays linearly over time (up to 90% or until minimum price is reached) until a winning bidder makes a purchase. All accumulated fees are distributed to the buyer.
* The purchase triggers the start of the next auction, where the starting price is set to be double the settled price.

### Key Auction Parameters

* **Starting Price**: Initial price set by admin for each auction (in cUSD)
* **Duration**: Auction duration set to 24 hours by default
* **Minimum Start Price**: Minimum starting price set to 100 cUSD
* **Payment Token**: cUSD is used as the payment token for all auctions
* **Recipient**: Fee Receiver contract receives all auction proceeds
* **Price Multiplier**: New auction starts at 2x the settled price of the previous auction

## Architecture

The Fee Auction system consists of three main components:

1. **FeeAuction Contract** - Facilitates Dutch auction mechanism for selling fees
2. **FeeReceiver Contract** - Manages fee distribution to stakeholders
3. **Gelato Automations** - Automates harvesting interest and distributing rewards

### Fee Auction

The [FeeAuction](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/feeAuction/FeeAuction.sol) contract implements a Dutch auction mechanism for selling accumulated yield.

#### Core Functions

**`buy`**: Allows permissionless purchase of accumulated fees at current auction price

```solidity
function buy(
    uint256 _maxPrice,address[] calldata _assets, uint256[] calldata _minAmounts, address _receiver, uint256 _deadline
) external;
```

* `_maxPrice`: Maximum price willing to pay
* `_assets`: Array of assets to purchase
* `_minAmounts`: Minimum amounts to purchase for each asset
* `_receiver`: Address to receive the purchased assets
* `_deadline`: Deadline for the auction purchase

**Admin Functions**

**`setStartPrice`**: Sets the starting price for the current auction

```solidity
function setStartPrice(uint256 _startPrice) external checkAccess(this.setStartPrice.selector)
```

**`setDuration`**: Sets the duration for future auctions

```solidity
function setDuration(uint256 _duration) external checkAccess(this.setDuration.selector)
```

**`setMinStartPrice`**: Sets the minimum starting price for future auctions

```solidity
function setMinStartPrice(uint256 _minStartPrice) external checkAccess(this.setMinStartPrice.selector)
```

**`setPaymentToken`**: Sets the payment token for the auction

```solidity
function setPaymentToken(address _paymentToken) external checkAccess(this.setPaymentToken.selector)
```

### Fee Receiver

The Fee Receiver distributes fees collected from the auction to stakeholders. The protocol can take a percentage of the accumulated yield for protocol treasury. The protocol fee is currently configured to 0.

#### Core Functions

**`distribute`**: Distributes fees to staked cap token holders

```solidity
function distribute() external;
```

The [`notify`](https://github.com/cap-labs-dev/cap-contracts/blob/25f3fc8a4aa186070381c7d19c170e524a815c5a/contracts/gelato/CapNotify.sol#L25) function is called when the vesting period is complete. The vesting period is configured to 24 hours.

#### Admin Functions

**`setCapToken`**: Sets the Cap token address

```solidity
function setCapToken(address _capToken) external checkAccess(this.setCapToken.selector)
```

**`setStakedCapToken`**: Sets the staked Cap token address

```solidity
function setStakedCapToken(address _stakedCapToken) external checkAccess(this.setStakedCapToken.selector)
```

**`setProtocolFeePercentage`**: Sets protocol fee percentage

```solidity
function setProtocolFeePercentage(uint256 _protocolFeePercentage) external checkAccess(this.setProtocolFeePercentage.selector)
```

**`setProtocolFeeReceiver`:** Sets protocol fee recipient

```solidity
function setProtocolFeeReceiver(address _protocolFeeReceiver) external checkAccess(this.setProtocolFeeReceiver.selector)
```

### Gelato Integrations

Gelato is used in the Fee Auction to automate yield harvesting and fee distribution via CapInterestHarvester and CapNotify respectively.

#### CapInterestHarvester

**`harvestInterest`**: Harvests yields from multiple sources and triggers fee distribution

```solidity
function harvestInterest() external;
```

Harvests yields from many sources via the [harvestInterest](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/gelato/CapInterestHarvester.sol#L61) function. The function executes the following:

1. **Fractional Reserve Harvesting**: Calls the Harvester contract to realize yields from FR strategies
2. **Lender Interest Claiming**: Claims accumulated interest from lending operations
3. **Flash Loan Optimization**: Uses Balancer flash loans to efficiently buy interest from fee auction
4. **Fee Distribution**: Triggers distribution of collected fees

**Execution Conditions**:

* Time-based: Execute if 24 hours have passed since last harvest
* Profit-based: Execute if minting cUSD and buying interest is profitable
* Gas Optimization: Only execute when profitable to cover gas costs

#### CapNotify

**`notify`**: Automates fee distribution to stcUSD holders

```solidity
function notify() external;
```

**Execution Conditions**:

* **Vesting Period**: Only execute after lock duration has passed
* **Time-based**: Respect profit vesting schedules
* **Staked CAP**: Ensure proper distribution to token holders

### Data Structures

#### FeeAuctionStorage

The FeeAuctionStorage stores auction configuration and state information.

```solidity
struct FeeAuctionStorage {
    uint256 startPrice;           // Starting price for current auction
    uint256 duration;             // Auction duration in seconds
    uint256 minStartPrice;        // Minimum starting price
    address paymentToken;         // Payment token (cUSD)
    address recipient;            // Fee receiver address
    uint256 auctionStartTime;     // Current auction start time
    uint256 currentPrice;         // Current auction price
    bool auctionActive;           // Whether auction is active
}
```

#### FeeReceiverStorage

The FeeReceiverStorage stores fee distribution configuration.

```solidity
struct FeeReceiverStorage {
    address capToken;                    // Cap token address
    address stakedCapToken;              // Staked cap token address
    uint256 protocolFeePercentage;       // Protocol fee percentage
    address protocolFeeReceiver;         // Protocol fee recipient
    uint256 vestingPeriod;               // Vesting period in seconds
    uint256 lastDistributionTime;        // Last distribution timestamp
    uint256 totalFeesDistributed;        // Total fees distributed
}
```
