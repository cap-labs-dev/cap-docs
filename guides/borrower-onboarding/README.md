# Borrower Onboarding

The following outlines the process for onboarding as an operator to borrow from Cap.

### 1. Requirements

* **Borrower Address**: A valid Ethereum address (EOA or multisig) to borrow
* **Underwriter**: Entity that provides credit for the Borrower. The list of current Underwriters can be found [here](https://cap.app/delegators).
* **Borrower Parameters:**
  * **Delegation Rate**: Fixed delegation premium rate on the amount borrowed
  * **LTV**: Loan-to-Value ratio for initial maximum borrow capacity
* **Protocol Whitelisting**: Contact the Cap team to request agent registration

{% hint style="info" %}
An Operator may only use one Ethereum address per Delegator and collateral pair. They must generate a new address for each new Delegator. Addresses cannot be changed after deployment. For best practices, we recommend using a multisig for the address.
{% endhint %}

### 2. Register to a SSN of choice

Currently, Cap supports the following Shared Security Networks.

1. [Symbiotic](symbiotic.md)
2. [EigenLayer](eigenlayer.md)

### 3. Complete Legal Agreements (Optional)

Operators and Delegators may enter into legal agreements outlining terms of delegation, responsibilities, and compliance.

### 4. Participate in Loan Activity

Once coverage is active from the respective shared security network, Operators can participate in borrowing activity directly from Cap's application using the Agent's address.

Review the risk parameters prior to loan:

* **LTV (Loan-to-Value)**: Maximum borrowing capacity (e.g., 50% = 0.5e27)
* **Maximum Borrow Amount:** Coverage x LTV
* **Liquidation Threshold**: Health factor at which liquidation can occur (e.g., 80% = 0.7e27)
* **LTV Buffer**: Minimum gap between LTV and liquidation threshold (10% = 0.05e27)

### 5. Updating Parameters

Should Operators wish to change loan parameters, please contact the Cap team to do so.

Namely, the restaker rate can be updated via the [setRestakerRate](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol) function, and the LTV and LT via the [modifyAgent](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol#L97) function.
