# Delegation

The Delegation module serves as a middleware between Shared Security Networks and the credit marketplace, enabling:

* **Borrower Management:** Register SSN Borrowers to Cap, setting their LTV/liquidation ratio and underwriting premiums
* **Collateral Provision**: Underwriters can collateralize Borrowers for borrow capacity
* **Liquidation:** Execute liquidation of delegated collateral during liquidation events
* **Reward Distribution**: Distribute rewards to Underwriters proportionally to their coverage

Cap supports multiple SSNs. Currently integrated SSNs include:

* [Symbiotic](symbiotic.md)
* EigenLayer

## Overview of Cap SSN Requirements

### Isolated Coverage

Delegation coverage is isolated both on a Network level and a Borrower level. Each Borrower is uniquely mapped to a SSN and an Underwriter. Each Borrower receives unique, isolated stake from a single Underwriter such that liquidation risk is not shared.

Network level isolation means that delegations cannot be "restaked" across multiple slashable restaking protocols — because delegations are used as collateral in Cap, a liquidation event in a different Network other than Cap would result in bad debt for Cap.&#x20;

Similarly, restaking across multiple Borrowers within Cap is not allowed. When multiple Underwriters are delegating to the same Borrower, a withdrawal from one of the Underwriters can result in a liquidation event for other Underwriters.&#x20;

In other words, an Underwriter's stake can only be used for a single Borrower in Cap. Alternatively, a Borrower cannot receive stake from multiple Underwriters. If the Borrower were to receive delegations from a new Underwriter, then the Borrower can simply create a new Ethereum address to do so.

### Underwriting Premiums

Each Borrower-Underwriter pair agrees upon a predetermined premium for underwriting. As such, each Borrower-Underwriter pair has a unique fixed yearly rate that can be updated via Admin controls. The rates are set in the respective SSN upon deployment.&#x20;

Underwriting premiums are distributed per each SSN's reward handler.&#x20;

### Liquidation and Redistribution

Liquidation conditions in Cap are objective: they are triggered when the health factor of the Borrower drops below the liquidation threshold. Liquidations can be called permissionlessly, and verified onchain. Liquidated funds are redistributed back to the Liquidators, ensuring cUSD is always backed 1:1.

To incentivize liquidations, liquidation and redistribution happen instantly. There are no veto committees nor delay periods for liquidations to occur, nor for the liquidated funds to be redistributed.&#x20;

### Whitelisted Participants

While the protocol can function fully autonomously, to prevent malicious actors from attacking the protocol, Cap will initially whitelist Underwriters and Borrowers. Underwriters and Borrowers that seek to join the protocol should refer to the Onboarding Guides to do so.

For function signatures, parameters, and data structures, see the [Delegation Contract Reference](../../developers/contracts/delegation.md).
