# Symbiotic

The following is the onboarding guide for Borrowers (referred to as _agents_ in Cap contracts). The onboarding flow has been mostly automated for the Borrower — the Underwriter can handle most bulk work when deploying an instance of the [Symbiotic Vault Factory](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/CapSymbioticVaultFactory.sol#L57) contract.

Steps:

1. Once agreements have been arranged with the Underwriter of choice, provide the Borrower's Ethereum address to the Underwriter. The Underwriter will then create the Borrower-specific Symbiotic Vault

* After receiving the delegations from the Underwriter, the Borrower is now eligible to borrow. Note, Delegations will take up to 6 days to be effective stake.

{% hint style="warning" %}
To isolate liquidation risk, each Borrower can only receive effective stake from one Symbiotic Vault. Delegations from other Vaults will not count as collateral. The Borrower must create a new address for each new Vault.
{% endhint %}

Under the hood, the following steps happen:

1. The Underwriter first creates a vault via the [CapSymbioticVaultFactory](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/CapSymbioticVaultFactory.sol#L57) including the Borrower address. The call will register the Borrower as SymbioticOperator in Symbiotic's [Operator Registry](https://docs.symbiotic.fi/modules/registries/#2-operatorregistry) and complete the [Operator to Network Opt-in](https://docs.symbiotic.fi/modules/registries/#2-operator-to-network-opt-in) and [Operator to Vault Opt-in](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/symbiotic/SymbioticNetwork.sol#L77) process via the [SymbioticOperator](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/SymbioticOperator.sol#L4) contract.
2. Once the Vault is deployed, Cap will register the Borrower to Cap's system via the [addAgent](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/interfaces/ISymbioticAgentManager.sol#L45) function. The function takes in the following struct:

```solidity
struct AgentConfig {
    address agent;
    address vault;
    address rewarder;
    uint256 ltv;
    uint256 liquidationThreshold;
    uint256 delegationRate;
}
```

This function will add the Borrower to the [Delegation](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol) contract, register the Vault and Borrower to Cap's [Symbiotic Network Middleware](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/SymbioticNetworkMiddleware.sol) and set underwriting premiums.

After stake becomes effective, Borrowers can start borrowing via the Asset page on Cap's app.

### Understanding Epochs

To be compatible with [Symbiotic's epoch model](https://docs.symbiotic.fi/learn/core-concepts/network#epoch), Cap has its own epoch system built. As a result, there are checkpoints where Cap reads stake from Symbiotic Vaults. The checkpoints are updated every 6 days, or when there is a borrow event.&#x20;

Functionally, this means that when Borrowers first receive stake, they cannot initiate a borrow until past the next Cap epoch.

### Verifying the Vault

Once the Symbiotic Vault is deployed, Borrowers may verify the deployment status via the [Delegation](https://etherscan.io/address/0x09A3976d8D63728d20DCDFEe1e531C206Ba91225#readProxyContract) and [Symbiotic Network](https://etherscan.io/address/0x98e52Ea7578F2088c152E81b17A9a459bF089f2a#readProxyContract) Contracts. Use the \_agent field to input the Borrower's Ethereum address.

* Delegation contract methods:
  * Collateral: delegation asset address
  * Coverage: how much stake is live
  * Vaults: the Borrower-Underwriter specific Symbiotic Vault
  * LTV: the max borrowable loan-to-value ratio
  * liquidationThreshold
* Symbiotic Network methods:
  * getOperator: get SymbioticOperator address for Borrower address

### Managing the Symbiotic App

Once registered, the Borrower profile will be updated in [Symbiotic's app page](https://app.symbiotic.fi/networks). If you wish to change the metadata on the app, please refer to [Symbiotic' guide](https://github.com/symbioticfi/metadata-mainnet) to do so.
