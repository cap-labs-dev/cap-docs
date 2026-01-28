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

## Architecture

The Fractional Reserve system consists of several interconnected components:

1. **FractionalReserve Contract** - Main logic for capital deployment
2. **TokenHolder** - ERC4626 Fractional Reserve Vault holding idle assets for strategy
3. **Harvester** - Orchestrates yield collection from all fractional reserve strategies
4. **LimitModule** - Restricts fractional reserve vault access to only the Cap vault
5. **Gelato Intergrations:**
   1. **CapSweeper:** Periodically deploys capital into the Fractional Reserve

### Fractional Reserves

The Fractional Reserve contract manages the deployment and withdrawal of idle capital into yield-generating strategies. The implementation logic can be found in the [FractionalReserveLogic](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/vault/libraries/FractionalReserveLogic.sol) contract.

#### Core Functions

**`investAll`**: Invests excess capital above reserve level to the ERC4626 vault. The excess capital is calculated as the difference between asset balance in the Vault and the reserve balance.

```solidity
function investAll(address _asset) external;
```

* `_asset`: Asset address to invest

**`divestAll`**: Divests all invested capital for an asset from fractional reserve strategies

```solidity
function divestAll(address _asset) external checkAccess(this.divestAll.selector)
```

* `_asset`: Asset address to divest

Key features:

* All of the Vault shares are redeemed from the Fractional Reserve Vault
* Supports [partial divests](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/vault/libraries/FractionalReserveLogic.sol#L84) in case of user withdrawals or emergency

**`realizeInterest`**: Withdraws accumulated interest from fractional reserve strategies&#x20;

```solidity
function realizeInterest(address _asset) external
```

* `_asset`: Asset address to realize interest for

Key features:

* Claimed interest is sent them to interest receiver ([Fee Auction](../fee-auction.md))
* Permissionless call

#### Admin Functions

**`setFractionalReserveVault`**: Sets the fractional reserve vault for an asset

```solidity
function setFractionalReserveVault(address _asset, address _vault) external checkAccess(this.setFractionalReserveVault.selector)
```

* `_asset`: Asset address
* `_vault`: New fractional reserve vault address

Key features:

* **Vault Migration**: Supports migration from old to new vault
* **Automatic Divestment**: Divests from old vault before setting new one

**`setReserve`**: Sets the reserve level for an asset

```solidity
function setReserve(address _asset, uint256 _reserve) external checkAccess(this.setReserve.selector)
```

* `_asset`: Asset address
* `_reserve`: Reserve level in asset decimals

#### View Functions

* **`claimableInterest`**: Get the amount of interest available to be realized
* **`fractionalReserveVault`**: Get the fractional reserve vault address for an asset
* **`fractionalReserveVaults`**: Get all fractional reserve vault addresses
* **`reserve`**: Get the reserve amount for an asset
* **`loaned`**: Get the loaned amount to the fractional reserve strategy for an asset
* **`interestReceiver`**: Get the interest receiver address

### Data Structures

#### FractionalReserveStorage

The FractionalReserveStorage stores all fractional reserve configuration and state.

```solidity
struct FractionalReserveStorage {
    address interestReceiver;                    // Interest receiver address
    mapping(address => uint256) loaned;         // Loaned amount per asset
    mapping(address => uint256) reserve;        // Reserve amount per asset
    mapping(address => address) vault;          // Fractional reserve vault per asset
    EnumerableSet.AddressSet vaults;            // Set of all fractional reserve vaults
}
```

### Token Holder

A TokenHolder Vault contains a simple strategy for holding the underlying asset. The implementation is inherited from Yearn V3's [Tokenized Strategy](https://github.com/yearn/tokenized-strategy). The Vault is deployed via the [TokenHolderFactory](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/fractionalReserve/TokenHolderFactory.sol).&#x20;

**Key features**

* Only allows the Cap vault to deposit/withdraw via the Limit Module
* Zero performance fees

**Core Functions**

* **`_deployFunds`**: Empty implementation (funds stay in contract)
* **`_freeFunds`**: Empty implementation (funds stay in contract)
* **`availableDepositLimit`**: Only allows Cap vault to deposit
* **`availableWithdrawLimit`**: Only allows Cap vault to withdraw

### Limit Module

The Limit Module provides access control for fractional reserve vault operations, ensuring only the Cap vault can interact with strategies

**Core Functions**

* **`available_deposit_limit`**: Only Cap vault can deposit
* **`available_withdraw_limit`**: Only Cap vault can withdraw

### Harvester

The Harvester contract harvests yield from fractional reserve vaults.

Specifically, for each strategy, the contract

* Calls [report](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/fractionalReserve/Harvester.sol#L20) on to record claimable interest
* Claims interest from the fractional reserves

**Key Functions**:

* **`harvest`**: Process all strategies and realize interest

### Gelato Integrations

#### CapSweeper

Automates capital deployment into fractional reserve strategies by calling investAll

**Configuration Parameters:**

* sweepInterval: 6 hours
* minSweepAmount: 10k cUSD

**Execution Conditions**:

* **Time Interval**: Respect minimum time between sweeps per asset
* **Asset Status**: Only sweep non-paused assets
* **Minimum Amount**: Only sweep if balance exceeds minimum threshold

## Assets and corresponding strategies

Currently, USDC is supported with the [Aave V3 lending strategy](https://github.com/cap-labs-dev/tokenized-aave-v3). More assets and strategies coming soon!

