# Symbiotic

The following is the onboarding guide for Operators (referred to as _agents_ in Cap contracts).  The process is automated for Operators by the Delegators via Cap's [Symbiotic Vault Factory](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/CapSymbioticVaultFactory.sol#L57) contract.&#x20;

Steps:

1. Once agreements have been arranged with the chosen Delegator, provide the Agent's Ethereum address to the Delegator. The delegator will then create the agent-specific Symbiotic Vault

* Delegator you wish to be receiving stake from: the delegator will then create the agent-specific Symbiotic Vault

{% hint style="warning" %}
To isolate slashing risk, each operator can only receive effective stake from one Symbiotic Vault. Delegations from other Vaults will not count as collateral. The operator must create a new address for each new Vault.&#x20;
{% endhint %}

Note, Delegations will take up to 6 days to be effective stake.

Under the hood, the following steps happen:

1. The Delegator first creates a vault via the [CapSymbioticVaultFactory](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/CapSymbioticVaultFactory.sol#L57) including the agent address. The call will register the operator in Symbiotic's [Operator Registry](https://docs.symbiotic.fi/modules/registries/#2-operatorregistry) and complete the [Operator to Network Opt-in](https://docs.symbiotic.fi/modules/registries/#2-operator-to-network-opt-in) and [Operator to Vault Opt-in](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/symbiotic/SymbioticNetwork.sol#L77) process via the [SymbioticOperator](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/SymbioticOperator.sol#L4) contract.&#x20;
2. Once the Vault is deployed, Cap will register the agent via to Cap's system via the [addAgent](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/interfaces/ISymbioticAgentManager.sol#L45) function. The function takes in the following struct:

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

This function will add the agent to the [Delegation](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol) contract, register the Vault and Agent to Cap's [Symbiotic Network Middleware](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/SymbioticNetworkMiddleware.sol) and set restaker rates

After stake become effective, Operators can start borrowing via the Asset page on Cap's app.&#x20;
