# Borrower Onboarding

The following outlines the process for onboarding as a Borrower to borrow from Cap.

### 1. Requirements

* **Borrower Address**: A valid Ethereum address (EOA or multisig) to borrow
* **Underwriter**: Entity that provides credit for the Borrower. The list of current Underwriters can be found [here](https://cap.app/underwriters).
* **Borrower Parameters:**
  * **Underwriting Premium**: Fixed premium rate on the amount borrowed
  * **LTV**: Loan-to-Value ratio for initial maximum borrow capacity
* **Protocol Whitelisting**: The Borrower must be whitelisted by the system.

{% hint style="info" %}
A Borrower may only use one Ethereum address per Underwriter and collateral pair. They must generate a new address for each new Underwriter. Addresses cannot be changed after deployment. For best practices, we recommend using a multisig for the address.
{% endhint %}

### 2. Register to a SSN of choice

Currently, Cap supports the following Shared Security Networks.

1. [Symbiotic](symbiotic.md)
2. [EigenLayer](eigenlayer.md)

### 3. Complete Legal Agreements (Optional)

Borrowers and Underwriters may enter into legal agreements outlining terms of delegation, responsibilities, and compliance.

### 4. Participate in Loan Activity

Once coverage is active from the respective shared security network, Borrowers can participate in borrowing activity directly from Cap's application using the Borrower's address.

Review the risk parameters prior to loan:

* **LTV (Loan-to-Value)**: Maximum borrowing capacity (e.g., 50% = 0.5e27)
* **Maximum Borrow Amount:** Coverage x LTV
* **Liquidation Threshold**: Health factor at which liquidation can occur (e.g., 80% = 0.8e27)
* **LTV Buffer**: Minimum gap between LTV and liquidation threshold (10% = 0.05e27)

{% hint style="info" %}
While the liquidation threshold is a global parameter, the LTV can be set by the Borrower-Underwriter Pair
{% endhint %}

### 5. Updating Parameters

To update the Underwriting Premium or the LTV for the Borrower, request a change to the Cap team.

Namely, the underwriting premium can be updated via the [setRestakerRate](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol) function, and the LTV and LT via the [modifyAgent](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/delegation/Delegation.sol#L97) function.
