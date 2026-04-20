# Delegation

The Delegation module serves as a middleware between Shared Security Networks and the lending protocol, enabling:

* **Operator Management:** Register SSN Operators to Cap, setting their LTV/liquidation ratio and restaker fees
* **Collateral Provision**: Delegators can collateralize Operators for borrowing capacity
* **Slashing:** Execute slashing of delegated collateral during liquidations
* **Reward Distribution**: Distribute rewards to delegators proportionally to their coverage

Cap supports multiple SSNs. Currently integrated SSNs include:

* [Symbiotic](symbiotic.md)
* EigenLayer

## Overview of Cap SSN Requirements

### Isolated Coverage

Delegation coverage is isolated both on a Network level and an Operator level. Each Operator is uniquely mapped to a SSN and a Delegator. Each Operator receives unique, isolated stake from a single Delegator such that slashing risk is not shared.

Network level isolation means that delegations cannot be "restaked" across multiple slashable restaking protocols - because delegations are used as collateral in Cap, a slashing event in a different Network other than Cap would result in bad debt for Cap.&#x20;

Similarly, restaking across multiple Operators within Cap is not allowed. When multiple Delegators are delegating to the same operator, a withdrawal from one of the Delegators can result in a slashing event for other Delegators.&#x20;

In other words, a Delegator's stake can only be used for a single Operator in Cap. Alternatively, an Operator cannot receive stake from multiple Delegators. If the Operator were to receive delegations from a new Delegator, then the Operator can simply create a new Ethereum address to do so.

### Restaker Fees

Each Operator-Delegator agree upon a predetermined fee for restaking. As such, each Operator-Delegator have an unique fixed yearly rate that can be updated via Admin controls. The rates are set in the respective SSN upon deployment.&#x20;

Restaker fees are distributed per each SSN's reward handler.&#x20;

### Slashing and Redistribution

Slashing conditions in Cap are objective: they are akin to liquidations and thus triggered when the health factor of the Operator drops below the liquidation threshold. Liquidations can be called permissionlessly, and verified onchain. Slashed funds are redistributed back to the liquidators, ensuring the cUSD is always backed 1:1.

To incentivize liquidations, slashing and redistribution happen instantly. There are no veto committees nor delay periods for slashing to occur, nor for the slashed funds to be redistributed.&#x20;

### Whitelisted Participants

While the protocol can function fully autonomously, to prevent malicious actors from attacking the protocol, Cap will initially whitelist Delegators and Operators. Delegators and Operators that seek to join the protocol should refer to the Onboarding Guides to do so.

For function signatures, parameters, and data structures, see the [Delegation Contract Reference](../../developers/contracts/delegation.md).
