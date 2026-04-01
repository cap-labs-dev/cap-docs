# Price Sources

This page describes **recommended ways to obtain an accurate cUSD price** (and similarly for stcUSD), plus how to derive pricing **on-chain** from the cUSD contract and the protocol oracle.

Contract addresses for Ethereum mainnet are listed in [Addresses](addresses.md).

## Why not use DEX / AMM pool prices?

Do **not** rely on on-chain DEX prices from venues such as Curve, Uniswap, or similar AMMs as your primary cUSD reference.

cUSD can be **minted and redeemed 1:1 with USDC** (and other whitelisted reserve assets at published rates). There is **little economic reason** for liquidity providers to maintain **deep** two-sided AMM liquidity when mint and redeem offer a direct, tight on/off-ramp. As a result, DEX pools are often **shallow**, and spot or TWAP prices there can be **noisy, easy to move, or unrepresentative** of the price implied by reserves and the protocol oracle. Prefer the feeds and oracle methods on this page instead.

## Recommended: external feeds

### RedStone (reserve oracle)

The RedStone feed used for cUSD reserve / fundamental pricing is the on-chain wrapper [`0x9A5a3c3Ed0361505cC1D4e824B3854De5724434A`](https://etherscan.io/address/0x9A5a3c3Ed0361505cC1D4e824B3854De5724434A) (Etherscan). The same feed is documented in the RedStone app at [https://app.redstone.finance/app/token/cUSD_FUNDAMENTAL/](https://app.redstone.finance/app/token/cUSD_FUNDAMENTAL/).

### DefiLlama (Coins API)

DefiLlama exposes current prices via the Coins API. Endpoint documentation: [https://api-docs.defillama.com/#tag/coins/get/prices/current/{coins}](https://api-docs.defillama.com/#tag/coins/get/prices/current/%7Bcoins%7D). Example for **cUSD** on Ethereum (checksum-insensitive in the path):

```
https://coins.llama.fi/prices/current/ethereum:0xcccc62962d17b8914c62d74ffb843d73b2a3cccc
```

For **stcUSD**, use the stcUSD token address on the same network, for example:

```
https://coins.llama.fi/prices/current/ethereum:0x88887be419578051ff9f4eb6c858a951921d8888
```

DefiLlama’s pegged-asset adapter for Cap cUSD (issued supply / peg logic) is [https://github.com/DefiLlama/peggedassets-server/blob/master/src/adapters/peggedAssets/cap-cusd/index.ts](https://github.com/DefiLlama/peggedassets-server/blob/master/src/adapters/peggedAssets/cap-cusd/index.ts).

## On-chain

The patterns below use the **cUSD** token contract and the mainnet **Oracle** (see [Addresses](addresses.md)).

### Backing balances per asset

Call `cUSD.assets()` to obtain the ordered list of backing assets, then for each `asset` call `cUSD.totalSupply(asset)` to read how much of that reserve token is held in the cUSD vault for that asset. Together, these balances describe the collateral side of the basket that backs outstanding cUSD.

### Redemption weights (1 cUSD → each underlying)

Using the same asset ordering from `cUSD.assets()`, call `cUSD.getRedeemAmount(1e18)` with **1e18** wei of cUSD (18 decimals). The return values give the amounts of each underlying you would receive for that redemption path, which you can treat as the per-asset conversion from one full cUSD unit into the basket (subject to fees and current oracle pricing encoded in the call). Align each entry in the returned arrays with the corresponding index in `assets()`.

### Protocol oracle price

Query the deployed **Oracle** contract ([0xcD7f45566bc0E7303fB92A93969BB4D3f6e662bb](https://etherscan.io/address/0xcD7f45566bc0E7303fB92A93969BB4D3f6e662bb)) with `getPrice(cusd_address)`, where `cusd_address` is the cUSD token address. That is the same `getPrice` surface as in [https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/interfaces/IPriceOracle.sol](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/interfaces/IPriceOracle.sol) (`IPriceOracle` in cap-contracts):

```solidity
function getPrice(address _asset) external view returns (uint256 price, uint256 lastUpdated);
```

For semantics (staleness, backup oracles, adapters), see [Oracles](../concepts/oracles.md).

## Etherscan proxy API

When trading volume on secondary-market pairs is low, a robust approach for both **cUSD** and **stcUSD** is to read the **protocol oracle** directly—sometimes described as using a **virtual price** (the economically meaningful value from the Oracle rather than thin AMM liquidity).

`getPrice(address _asset)` returns **two** `uint256` values: the **price** and **`lastUpdated`** (last update timestamp). The signature is specified in Cap’s [`IPriceOracle`](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/interfaces/IPriceOracle.sol#L51) (`getPrice`).

**Integrator note:** If you only deploy a small utility contract that forwards `getPrice` but **drops** `lastUpdated`, consumers may still get a spot price, but **historical** pricing pipelines often need **both** fields (or an equivalent archive of oracle updates) to attribute values correctly in time. Where possible, decode the full ABI return from `eth_call` against the Oracle (or index oracle update events) instead of stripping the timestamp.

### Example: `eth_call` via Etherscan API v2

These are illustrative `module=proxy` `eth_call` URLs for Ethereum mainnet (`chainid=1`). Replace `YOURAPIKEY` with a valid [Etherscan API](https://docs.etherscan.io/) key.

**cUSD** (`getPrice` on the Oracle for the cUSD token address):

```
https://api.etherscan.io/v2/api?chainid=1&module=proxy&action=eth_call&to=0xcD7f45566bc0E7303fB92A93969BB4D3f6e662bb&data=0x41976e09000000000000000000000000cccc62962d17b8914c62d74ffb843d73b2a3cccc&tag=latest&apikey=YOURAPIKEY
```

**stcUSD:**

```
https://api.etherscan.io/v2/api?chainid=1&module=proxy&action=eth_call&to=0xcD7f45566bc0E7303fB92A93969BB4D3f6e662bb&data=0x41976e0900000000000000000000000088887be419578051ff9f4eb6c858a951921d8888&tag=latest&apikey=YOURAPIKEY
```

Example JSON-RPC style response (shape may vary slightly by client; `result` is the ABI-encoded return data):

```json
{"jsonrpc":"2.0","id":1,"result":"0x0000000000000000000000000000000000000000000000000000000005f56ea50000000000000000000000000000000000000000000000000000000069ca971b"}
```

Decode as two 32-byte words: first word = `price`, second word = `lastUpdated` (Unix timestamp in seconds). Scaling and staleness rules for `price` follow the Oracle configuration described in [Oracles](../concepts/oracles.md).
