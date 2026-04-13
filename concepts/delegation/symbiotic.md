# Symbiotic

Cap integrates Symbiotic's restaking infrastructure to create a delegation-based lending system where delegators provide coverage for Operators (agents). Delegators can deploy Vaults to provide stake to Operators as coverage, where depositors can deposit and withdraw out of the vault.

## Overview

### Vault to Operator relationship

* **Unique Operator-Vault pair**: Each Symbiotic Vault can only delegate to one Operator. An operator must use a different Ethereum address in order to receive delegations from a new Vault. Likewise, a Delegator must deploy a new Vault in order to delegate to a new operator.
* Once an Operator starts receiving delegations, the Operator address is immutable.

### Lifecycle of Symbiotic Vault

1. Vault Creation: Delegator creates a Vault via the Cap Symbiotic Vault Factory Contract, which accepts one ERC20 collateral type
2. Cap Whitelisting: After the Vault is deployed, Cap adds the Vault and Operator address to Cap system along with loan parameters and restaker rates
3. Vault Management: Once the Operator-Vault pair is live, delegators have access control over depositors and claiming rewards. Liquidations will trigger slashing events on the Vault.

Let's dive deeper into each of the steps.

## Vault Creation

### Deploy Vault

<figure><img src="../../.gitbook/assets/image (29).png" alt=""><figcaption><p>Cap Symbiotic Vault creation and deposit flow</p></figcaption></figure>

The `createVault` function is the core deployment function in the `CapSymbioticVaultFactory` contract.

{% hint style="info" %}
Vaults that are not registered using the Vault Factory will not be accepted in Cap's system.
{% endhint %}

```solidity
function createVault(
    address _owner,      // Vault owner/admin address
    address _asset,      // Collateral asset address (e.g., wstETH)
    address _agent,      // Agent address for delegation coverage
    address _network     // Cap Symbiotic network address
) external returns (
    address vault,       // Deployed vault contract address
    address delegator,   // Deployed delegator contract address
    address burner,      // Deployed burner router address
    address slasher,     // Deployed slasher contract address
    address stakerRewards // Deployed staker rewards contract address
);
```

Specifically, the function executes the following:

1. deploys a Symbiotic operator for the specified agent to manage delegation
2. deploys Symbiotic Vault and modules according to Cap requirements

Let us dive deeper into the specific requirements of Cap Symbiotic Vaults.

#### 1. Burner

The burner specifies where assets are transferred to when slashing happens.

A burner router is deployed to set the receiver of the slash. By setting the owner of the router to be the zero address, the global receiver is immutably set to be Cap's middleware.

```solidity
function _deployBurner(address _collateral) internal returns (address) {
    return burnerRouterFactory.create(
        IBurnerRouter.InitParams({
            owner: address(0),                    // No owner (immutable)
            collateral: _collateral,              // Asset to burn
            delay: 0,                            // Instant execution
            globalReceiver: middleware,          // Cap network middleware
            networkReceivers: new IBurnerRouter.NetworkReceiver[](0),
            operatorNetworkReceivers: new IBurnerRouter.OperatorNetworkReceiver[](0)
        })
    );
}
```

#### 2. Delegator

The Delegator specifies whether restaking is allowed across networks and operators, and the allocation of assets.

Cap requires that staked assets are solely used as coverage for the specific operator receiving delegations (i.e. stake cannot be shared with other networks/operators). As such, the [OperatorNetworkSpecificDelegator](https://docs.symbiotic.fi/modules/vault/delegation#3-operatornetworkspecificdelegator) is used to ensure delegations are siloed. Each vault can only delegate to one operator.

#### 3. Slasher

The Slasher handles slash requests from Cap’s middleware, by fetching stake from the Delegator and calling the Vault to transfer the assets to the Burner.

The vault uses `INSTANT` slasher type for immediate slash execution, so that liquidation bonuses can be redistributed immediately.

#### 4. Vault

The Vault handles deposit and withdrawals on an epoch basis. Assets leave Symbiotic vaults only when there is a withdrawal or slashing event.

Deposits are instant. Withdrawals take until the end of the next epoch to withdraw. The epoch is fixed to 7 days, hence withdrawals take up to 14 days to execute.

#### 5. Staker Rewards

The stakerRewards contract creates a rewards contract to distribute interest to delegators

```solidity
stakerRewards = defaultStakerRewardsFactory.create(
    IDefaultStakerRewards.InitParams({
        vault: vault,
        adminFee: 0,                             // Initial admin fee (0%)
        defaultAdminRoleHolder: _owner,          // Vault owner gets admin role
        adminFeeClaimRoleHolder: _owner,         // Vault wner can claim admin fees
        adminFeeSetRoleHolder: _owner            // Vault owner can set admin fees
    })
);
```

### Whitelisting

After the Vault is created, the Vault needs to be added to Cap's system.

Cap whitelists the Agent-Vault pair via the addAgent function in the [SymbioticAgentManager](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/providers/symbiotic/SymbioticAgentManager.sol#L42) contract. The contract acts as a bridge between the Cap delegation system and the Symbiotic restaking infrastructure, ensuring proper registration and configuration of agents in the system.

The function takes in the following parameters:

```solidity
struct AgentConfig {
    address agent;              // Agent address (operator)
    address vault;              // Associated Symbiotic vault
    address rewarder;           // Staker rewards contract
    uint256 ltv;                // Loan-to-value ratio (e.g., 0.5e27 = 50%)
    uint256 liquidationThreshold; // Liquidation threshold (e.g., 0.7e27 = 70%)
    uint256 delegationRate;     // Delegation rate for interest calculation
}
```

First, the pair is added to the delegation contract, configuring loan parameters such as LTV and LT. By default, LTV is set to 50%, with the liquidation threshold at 80%. Notice, the delegation rate is also configured in this step.

```solidity
// Add agent to delegation contract
IDelegation(delegation).addAgent(
    agentAddress,    // Agent address
    middleware,      // Network middleware
    0.5e27,         // LTV (50%)
    0.8e27          // Liquidation threshold (80%)
);
```

Next, the Vault, rewarder and agent are registered to Cap's Middleware, completing necessary Symbiotic Opt In processes.

```solidity
ISymbioticNetworkMiddleware($.networkMiddleware).registerVault(
    _agentConfig.vault,
    _agentConfig.rewarder,
    _agentConfig.agent
);
```

In particular, a subnetwork identifier is created for the operator. The identifier is used to enforce the one-to-one relationship between the agent and the vault. Delegations to other subnetwork identifiers will not count as effective stake.

## Vault Management

### Rewards

**`distributeRewards`**: Distributes rewards through Symbiotic network

```solidity
function distributeRewards(address _agent, address _token) external checkAccess(this.distributeRewards.selector)
```

* `_agent`: Agent address for reward distribution
* `_token`: Reward token address

### Slashing

**`slash`**: Executes slashing through Symbiotic network via the Burner

```solidity
function slash(address _agent, address _recipient, uint256 _slashShare, uint48 _timestamp) external checkAccess(this.slash.selector)
```

* `_agent`: Agent address to slash
* `_recipient`: Address to receive slashed collateral
* `_slashShare`: Share of collateral to slash
* `_timestamp`: Timestamp for slashing

### Admin Control

* `DEFAULT_ADMIN_ROLE` - Full vault management
* `DEPOSIT_WHITELIST_SET_ROLE` - Manage deposit whitelist
* `DEPOSIT_LIMIT_SET_ROLE` - Set deposit limitst
