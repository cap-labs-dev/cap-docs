# Fee Auction

The [Fee Auction](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/feeAuction/FeeAuction.sol) facilitates permissionless Dutch auctions where collected protocol fees are sold to the winning bidder. The proceeds are sent to the [Fee Receiver](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/feeReceiver/FeeReceiver.sol) to convert to cUSD and distribute to stcUSD holders. The Fee Auction module provides an efficient and transparent way to distribute accumulated fees to participants while ensuring fair price discovery through time-based price decay.&#x20;

## Mechanics

### Overview of Operations

1. **Interest Harvesting**: Interest Harvester realizes accumulated interest and sends it to Fee Auction
2. **Dutch Auction**: Fee Auction sells accumulated assets via a Dutch auction mechanism
3. **Fee Distribution**: Fee Receiver collects cUSD from auction sales and distributes to stcUSD token holders

<figure><img src="../.gitbook/assets/image (33).png" alt=""><figcaption><p>Flow of Funds: Fee Auction</p></figcaption></figure>

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

For function signatures, parameters, and data structures, see the [Fee Auction Contract Reference](../developers/contracts/fee-auction.md).
