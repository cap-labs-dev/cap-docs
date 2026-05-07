# EigenLayer

The following is the onboarding guide for Delegators using EigenLayer. The process is fully automated by the Cap system via the [EigenAgentManager](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenAgentManager.sol) contract.

### 1. Submit AgentConfig information to the Cap team&#x20;

This is the only action the Delegator needs to take. The step can be completed by the Operator (Agent). The rest of the process is handled on behalf of the Delegator.

```solidity
struct AgentConfig {
    address agent;              // Your agent/borrower address
    address strategy;           // EigenLayer strategy address (e.g., wstETH)
    address restaker;           // Address of the restaker who will delegate to you
    string avsMetadata;         // AVS metadata URI (can be empty string)
    string operatorMetadata;    // Operator metadata URI (can be empty string)
    uint256 ltv;               // Loan-to-Value ratio (e.g., 5e26 for 50%)
    uint256 liquidationThreshold; // Liquidation threshold (e.g., 8e26 for 80%)
    uint256 delegationRate;    // Delegation rate for rewards (e.g., 2e25 for 2%)
}
```

If the strategy does not exist for the collateral asset, it can be permissionlessly created from the [StrategyFactory](https://github.com/Layr-Labs/eigenlayer-contracts/blob/8e25c9da31d5bb92de4fda14164d73911207aad5/src/contracts/strategies/StrategyFactory.sol#L4) contract.

{% hint style="warning" %}
The Delegator cannot delegate to multiple Operators. A new address must be used to delegate to a new Operator.
{% endhint %}

### 2. Agent Registration

Cap's Admin registers the Agent to both EigenLayer and Cap via the [addEigenAgent](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenAgentManager.sol#L44) function.&#x20;

Specifically, the function performs the following:

* Registers Agent to Cap via [addAgent](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/interfaces/ISymbioticAgentManager.sol#L45) function in the Delegation contract
* Deploys [EigenOperator](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenOperator.sol) beacon proxy which handles
  * Registering to EigenLayer's DelegationManager
  * Creating and registering to a new Operator Set in EigenLayer's AllocationManager
  * Setting Operator AVS reward split 0%
  * Allowlisting TOTP digest for Delegators (valid for 28 days)

### 3. Deposit

At this time, Delegators can deposit into the Strategy to receive strategy shares via the [depositIntoStrategy](https://github.com/Layr-Labs/eigenlayer-contracts/blob/8e25c9da31d5bb92de4fda14164d73911207aad5/src/contracts/core/StrategyManager.sol#L80) function.&#x20;

### 4. Allocation Delay

After registration, there is a **17.5** day delay of the allocation configuration for EigenLayer. During this delay, any delegations will **not** count as effective collateral. This is because the allocation magnitude cannot be updated prior to this allocation configuration delay period.

After 17.5 days, Operators can allocate delegated collateral via the permissionless [allocate](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenServiceManager.sol#L227) function.

### 5. Delegate

Once Operators allocate, strategy shares can be delegated to the EigenOperator via the [delegateTo](https://github.com/Layr-Labs/eigenlayer-contracts/blob/8e25c9da31d5bb92de4fda14164d73911207aad5/src/contracts/core/DelegationManager.sol#L124) function in the [DelegationManager](https://github.com/Layr-Labs/eigenlayer-contracts/blob/8e25c9da31d5bb92de4fda14164d73911207aad5/src/contracts/core/DelegationManager.sol) contract. Delegations will immediately count as effective coverage.&#x20;

#### TOTP

When the addEigenAgent function is called, it creates a TOTP digest that is used when delegating to the EigenOperator. The TOTP delegation approval digest has an expiry timestamp that can be queried by the [getCurrentTotpExpiryTimestamp](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenOperator.sol#L148) function. If expired, a new digest can be generated via the [advanceTotp](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenOperator.sol#L105) function.

With live coverage, the onboarding process is complete. Operators can now borrow against delegated collateral.

### 6. Verifying Information

* Agent Registration:
  * IDelegation(delegation).agentData(agent).network should return EigenServiceManager's address
* Agent Config:
  * IDelegation(delegation).agentData(agent)&#x20;
* EigenOperator Deployment:
  * IEigenServiceManager(eigenServiceManager).getEigenOperator(agent)
  * Operator address:
    * IEigenOperator(eigenOperator).operator()
  * Delegator address:
    * IEigenOperator(eigenOperator).restaker()
* Current Coverage:
  * IEigenServiceManager(eigenServiceManager).coverage(agent)
* Delegated Shares:
  * IDelegationManager(delegationManager).getDepositedShares(restakerAddress)
* TOTP expiry:
  * IEigenOperator(eigenOperator).getCurrentTotpExpiryTimestamp()

### 7. Tracking Rewards

When the agent borrows, interest is paid and distributed to restakers through EigenLayer's RewardsCoordinator. Please refer to EigenLayer's [docs](https://docs.eigencloud.xyz/eigenlayer/concepts/rewards/rewards-claiming) to understand how rewards can be claimed.
