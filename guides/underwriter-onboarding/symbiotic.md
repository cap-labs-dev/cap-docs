# Symbiotic

## Overview

Vault curators on Symbiotic can easily deploy Cap-specific Symbiotic Vaults using the [CapSymbioticVaultFactory](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/symbiotic/CapSymbioticVaultFactory.sol) directly on the app's [Create Vault page](https://cap.app/delegators/create-vault). Cap must first approve the Vault before delegated stake can be used as collateral.

### 1. Deploy Symbiotic Vault

Curators can deploy Cap-specific Symbiotic Vaults either via

1. Cap's [UI](https://cap.app/underwriters/create-vault) for creating Vault
2. Vault Factory contract's [<kbd>createVault</kbd>](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/CapSymbioticVaultFactory.sol#L57) function on Etherscan.&#x20;

As most of the parameters and modules are preconfigured, curators only have to specify the Borrower address and Collateral asset address when creating the Vault.

The factory contract will create Symbiotic's Delegator, Burner, Slasher and Rewarder modules as needed in Cap's Symbiotic Vault requirements.

```solidity
/// @param _owner The owner of the vault, will manage delegations and set deposit limits 
/// @param asset The asset of the vault 
/// @param _agent The agent of the vault 
/// @param _network The network of the vault 
/// @return vault The address of the new vault function 
createVault(address _owner, address asset, address _agent, address _network) external 
returns (address vault, address delegator, address burner, address slasher, address stakerRewards); 
```

{% hint style="info" %}
To ensure that all parameters are set-up as intended in Cap's design, Cap only accepts Vaults deployed via the factory contract.&#x20;
{% endhint %}

Once deployed,the Borrower-Underwriter pair will be added to Cap's contracts via SymbioticAgentManager's [`addAgent`](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/SymbioticAgentManager.sol#L42) function. The function handles necessary registry of the Vault. Once this part is complete, Underwriters can start using delegated stake as collateral.&#x20;

{% hint style="info" %}
Vault creation will revert if the Borrower's address is already receiving delegations from another Vault.
{% endhint %}

### 2. Admin Controls

Key administrative functions available to vault admins include configuring admin fees, setting deposit limits, and whitelisting depositors. Admin fees are taken from the rewards distributed to depositors, as a percentage of the total rewards.&#x20;

```solidity
// Set an admin fee
stakerRewards.setAdminFee(adminFee);

// Enable/disable deposit limit
vault.setIsDepositLimit(status);

// Set deposit limit
vault.setDepositLimit(amount);

// Enable/disable deposit whitelist
vault.setDepositWhitelist(status);

// Add/remove whitelisted depositors
vault.setDepositorWhitelistStatus(depositor, status);
```

### 3. Collecting Rewards

Rewards are automatically distributed to the StakerRewarder when a Borrower repays a loan.

Underwriters can also manually claim rewards via the <kbd>realizeRestakerInterest</kbd> function on the [Lender](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/lendingPool/Lender.sol) contract. The contract will distribute accrued interest on the Borrower's borrowed amount to the StakerRewarder.&#x20;

Vault admins can claim the admin fee via the <kbd>claimAdminFee</kbd> function of the Symbiotic StakerRewards.

### 4. Managing Withdrawals

Withdrawals from Symbiotic Vaults take up to 2 epochs to process. Specifically, withdrawals start after the current epoch plus another epoch. Since an epoch is 7 days for Cap Symbiotic Vaults, it takes 8-14 days to complete the withdrawal.

While the withdrawal is processed, the delegation assets are still liquidatable. If the withdrawal triggers an unhealthy position for the Borrower, then the asset may be liquidated (after the grace period). It is thus crucial that the withdrawal amount does not affect the Borrower's position: it is best advised for Underwriters to coordinate the withdrawal with the Borrower.


### 5. Managing the Symbiotic App
Once registered, the Underwriter profile will be updated in [Symbiotic's app page](https://app.symbiotic.fi/networks). If you wish to change the metadata on the app, please refer to [Symbiotic' guide](https://github.com/symbioticfi/metadata-mainnet) to do so.
