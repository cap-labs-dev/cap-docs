# Fractional Reserves

The [Fractional Reserve](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/vault/FractionalReserve.sol) contracts allows the Vault to deploy idle capital into yield-generating strategies per reserve asset. In order to ensure sufficient liquidity for immediate withdrawals and redemptions, a fixed amount of capital can be set in the buffer reserve.&#x20;

The yield strategies are restricted to safe, verifiable sources such as direct revenue sharing from the reserve assets, or deployment into leading crypto lending markets such as Aave. This design ensures capital efficiency of the underlying assets while maintaining redeemability guarantees and reserve transparency.

[Gelato](https://www.gelato.cloud/web3-functions) is used in Cap's Fractional Reserve system to automate capital deployment, yield harvesting, and fee distribution.

## Mechanics

<figure><img src="../../.gitbook/assets/image (2) (1).png" alt=""><figcaption><p>Flow of Funds: Fractional Reserves</p></figcaption></figure>

The flow is as follows:

1. cUSD minters deposit assets into Cap Vault
2. Excess capital is invested into Fractional Reserve Vaults, where TokenHolder strategies will generate passive yield
3. Accrued yield is sent to the Fee Auction, sold for cUSD
4. The cUSD is transferred to the Fee Receiver which then periodically distributes rewards to stcUSD holders

Fractional Reserve Vault & Gelato Mechanics

* A ERC4626 [TokenHolder](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/fractionalReserve/TokenHolder.sol) Fractional Reserve Vault is deployed for the underlying reserve asset (i.e. USDC) that acts as a strategy following Yearn V3's [Tokenized Strategy](https://github.com/yearn/tokenized-strategy) pattern. (i.e. lending to Aave V3). Only the Fractional Reserve Vault can deposit and withdraw from the Vault.
* The [CapSweeper](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/gelato/CapSweeper.sol) contract automatically invests excess assets every 6 hours
* The strategy earns interest in the asset supplied until divested. [CapInterestHarvester](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/gelato/CapInterestHarvester.sol) is used to automate yield harvesting from the fractional reserve strategies to the Fee Auction

### Key Fractional Reserve Parameters

* **Reserve Level**: Minimum amount of each asset to keep in the vault (not invested)
* **Loaned Amount**: Total amount of each asset currently invested in fractional reserve strategies
* **Interest Receiver**: Address that receives realized interest (set to [Fee Auction](../fee-auction.md))
* **Claimable Interest**: Amount of interest available to be realized from strategies
* **Investment Threshold**: Minimum amount required to invest in strategies

For function signatures, parameters, and data structures, see the [Fractional Reserve Contract Reference](../../developers/contracts/fractional-reserve.md).

## Assets and corresponding strategies

Currently, USDC is supported with the [Aave V3 lending strategy](https://github.com/cap-labs-dev/tokenized-aave-v3). More assets and strategies coming soon!

