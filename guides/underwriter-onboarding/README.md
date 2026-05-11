# Underwriter Onboarding

The following outlines the process for onboarding Underwriters to Shared Security Networks (SSNs) in Cap's protocol. Onboarded Underwriters may delegate to Borrowers approved in the Cap system.

### 1. Setup

Ensure you have:

* **Borrower Address:** Borrower's Ethereum address that will receive delegations. The Borrower must be already registered in Cap's system.
* **Underwriting Premium**: Agree on a fixed rate to receive from the Borrower
* **Collateral Asset:** The asset to be used as vault collateral (ETH/BTC-denominated ERC20s)

### 2. Choose SSN

Select SSN of choice for delegations, and follow the corresponding onboarding guide to complete set up and start delegating.

1. [Symbiotic](symbiotic.md)
2. [EigenLayer](eigenlayer.md)

Collateral management, i.e. delegations and withdrawals, are handled within each SSN. Underwriters are advised to monitor delay periods specific to the SSN.

{% hint style="danger" %}
Withdrawing delegations immediately lowers coverage. Delegated assets in the withdrawal queue are liquidatable until the end of the withdrawal delay. Hence, a withdrawal below the liquidation threshold makes the delegation asset liquidatable.&#x20;
{% endhint %}

While a time buffer is in place to mitigate risk, it is recommended that Underwriters whitelist depositors to prevent malicious/accidental withdrawals that may trigger unintended liquidation.

### 3. Complete Legal Agreements (Optional)

Borrowers and Underwriters may enter into legal agreements outlining terms of delegation, responsibilities, and compliance.

### 4. Updating Parameters

Should Borrowers wish to change loan parameters, please contact the Cap team to do so.

Namely, the underwriting premium can be updated via the [setRestakerRate](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol) function, and the LTV and LT via the [modifyAgent](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol#L97) function.
