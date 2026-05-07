# EigenLayer

The following is the onboarding guide for Agents (Operator) using EigenLayer. The process is fully automated by the Cap system via the [EigenAgentManager](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenAgentManager.sol) contract.

### 1. Submit AgentConfig information to the Cap team&#x20;

This is the only action an Operator needs to take. The step can be completed by the Operator.  The rest of the process is handled on behalf of the Operator.

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

{% hint style="warning" %}
The Operator must use a new address for each new Delegator. The Operator **cannot** delegate to other AVSs.
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

Once registered, you will have an EigenOperator proxy address and an Operator Set ID for the Cap AVS.

### 3. Allocation

After registration, there is a **17.5** day delay of the allocation configuration for EigenLayer. In this time, Delegators cannot delegate to the Eigen Operator and as such, Operators cannot participate in loan activities. This is because allocation magnitude cannot be updated prior to this allocation configuration delay period.

After the delay period, Operators can allocate delegated collateral to Cap so that it counts as effective coverage, via the permissionless [allocate](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenServiceManager.sol#L227) function. Adding and withdrawing collateral after this time is now instant.

Once coverage is live, Operators can now borrow against delegated collateral.

{% hint style="info" %}
If allocation or delegation fails, make sure the TOTP is updated via the [advanceTotp](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenOperator.sol#L105) function.
{% endhint %}

### 4. Verifying Information

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
* TOTP expiry:
  * IEigenOperator(eigenOperator).getCurrentTotpExpiryTimestamp()

### 5. Updating Metadata

Operators can update operator metadata at any time via the [updateOperatorMetadataURI](https://github.com/cap-labs-dev/cap-contracts/blob/main/contracts/delegation/providers/eigenlayer/EigenOperator.sol#L98). Please reach out to the Cap team if you are unable to host the metadata.
