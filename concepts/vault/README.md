# Vault

The Vault is the core module responsible for the storage, issuance, and redemption of cUSD, and for managing the underlying collateral assets. It plays a pivotal role as the protocol's liquidity backbone by enabling minting and burning of cUSD, facilitating liquidity for borrowers, and ensuring capital efficiency through the Fractional Reserves.

## Overview of operations

* **Mint/Burn:** cUSD can be minted/burned _\~1:1_ for any of the supported backing assets. Fees are dynamically calculated according to the allocation ratio of the asset.&#x20;
* **Redeem:** cUSD can be redeemed for a proportional amount of the basket of assets for a fixed fee.
* [**Fractional Reserves**](fractional-reserves.md): Idle capital in the Vault is deployed into yield-generating strategies (US T-bill yield, crypto lending markets) until borrowed or withdrawn.
* **Borrow/Repay**: When operators borrow/repays assets, the Vault tracks utilization of the pool

## Mechanics

### Mint/Burn

The Vault is the main interface for liquidity providers to mint and burn cUSD. cUSD can be minted/burned at Oracle value for any of the assets. Fees are calculated in the [Minter](minter.md) contract according to the allocation of the assets.

When a user mints/burns, the flow is as follows:

1. **User Call**: User calls `Vault.mint()` (or burn) with asset and amount
2. **Fee Calculation**: Vault calls `Minter.getMintAmount()` (or getBurnAmount) to calculate fees
3. **Logic Processing**: MinterLogic calculates amount out and fees at current Oracle value
4. **State Update**: VaultLogic handles asset transfer and state updates
5. **Token Minting**: cUSD is minted/burned in exchange for underlying assets

{% hint style="info" %}
Mint and burn functions are disabled if Oracle prices are stale, until Oracles are back in sync
{% endhint %}

### Redeem

A depeg event of any of the underlying assets can result in a last man standing problem where the last to withdraw will be left with the depegged asset. When such events occur, the protocol incentivizes users to withdraw proportionally to the ratio of the assets, effectively socializing the losses. Redeem fees are charged at a fixed percentage, whereas burn fees are dynamic, making it economically infeasible to burn for USDT beyond the burn kink ratio. For instance, if the basket contains 50% of USDC valued at $0.9 and 50% of USDT valued at $1, then the redeem request for $100 cUSD should withdraw $45 worth of USDC and $50 worth of USDT minus fees.&#x20;

When a user redeems, the flow is as follows:

1. **User Call**: User calls `Vault.redeem()` with cUSD
2. **Fee Calculation**: Vault calls `Minter.getRedeemAmount()` to calculate proportional amounts
3. **State Update**: cUSD is burned from user, assets are divested from strategies
4. **Asset Transfer**: All underlying assets are transferred to user

### Borrow Operations

Registered Borrowers can borrow assets from the Vault via the Lender, provided they are sufficiently collateralized by delegated assets from shared security networks. While assets are lent out from the Vault, the core borrow logic is implemented in the Lender contract. When the Borrower requests to borrow via the Lender, the Lender calls `Vault.borrow()`  to transfer assets to the Lender, updating the utilization and reserve state of the Vault.&#x20;

Similarly, Borrowers can repay via the Lender, which transfers borrowed assets back to the Vault.

### Vault Management

Assets must first be whitelisted in order to be used in Cap. Only regulated dollar-denominated crypto assets are accepted as a backing assets.

Interest rates for borrowing each asset of the reserve is a function of the utilization rate of the asset. The [current utilization rate](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/vault/Vault.sol#L198) of each asset can be queried in the Vault contract.

### Configurations

* **Whitelist minters**: Supports fee bypass for privileged users
* **Pause mechanisms**:
  * Asset-Level Pause: Individual assets can be paused while others remain active
  * Protocol-Level Pause: Complete protocol shutdown capability

### Key Vault Parameters

* **Total Supplies**: Total amount of each asset deposited in the vault
* **Total Borrows**: Total amount of each asset borrowed from the vault
* **Utilization Rate**: Ratio of borrowed assets to total supplies, calculated as (Total Borrows / Total Supplies) \* 100%
* **Available Balance**: Amount of asset available for borrowing, calculated as Total Supplies - Total Borrows
* **Utilization Index**: Cumulative utilization tracking for interest rate calculations
* **Insurance Fund**: Address that receives fees from mint/burn operations
* **Pause States**: Individual asset pause states and protocol-wide pause capability

For function signatures, parameters, and data structures, see the [Vault Contract Reference](../../developers/contracts/vault.md).
