# Borrow

Whitelisted Borrowers in Cap can borrow and repay the underlying Vault assets, provided they have secured sufficient collateral from an Underwriter.&#x20;

## Mechanics

### Borrow

The borrow process flow is as follows:

1. Borrower calls the <kbd>borrow</kbd> function on the Lender contract
2. Underwriter interest is realized first to prevent charging for compounded interest
3. If eligible to borrow\*, assets are lent out from the Vault to the Borrower
4. Debt tokens are minted to track the loan&#x20;

**Key Validations\***:

* Borrower must have sufficient collateral and health factor
* Borrower must be whitelisted and not paused
* Asset being borrowed should not be paused
* Borrow amount must meet minimum requirements, and may not exceed maximum borrows
  * Minimum: set by Admin via the setMinBorrow function
  * Maximum: The smaller value of the Borrower's remaining borrow capacity, and the remaining available amount to be borrowed from the Vault

### Repay

The repay process flow is as follows:

1. Borrower calls the <kbd>repay</kbd> function on the Lender contract
2. Underwriter interest is realized first to ensure all interest is accounted for
3. The repayment is processed in the following order:
   1. Unrealized underwriting premium (if any)
   2. Vault principal debt
   3. Vault interest
4. Debt tokens corresponding to the total amount repaid are burned

{% hint style="info" %}
If the repayment is not in full, the system will maintain a minimum borrow amount for the Borrower.&#x20;
{% endhint %}

#### Interest Distribution

The repaid assets are distributed back to the shareholders:&#x20;

1. **Vault Principal**: Sent to [Vault](../vault/) contract's reserves
2. **Vault Interest**: Sent to `interestReceiver` (set to [Fee Auction](../fee-auction.md)). Any excess interest will go to the interest receiver
3. **Underwriting Premium**: Sent to [Delegation](../delegation/) contract for distribution to Underwriters
4. **Unrealized Interest**: Added to Borrower's debt balance for future repayment

### Realizing Interest

Interest in Cap is **accrued continuously** but **realized discretely**:

* **Accrued Interest**: Calculated in real-time based on elapsed time and rates
* **Realized Interest**: Actually borrowed from vault and distributed to recipients
* **Unrealized Interest**: Accrued but not yet realized due to vault liquidity constraints

As can be seen, interest can be realized prior to repayment in Cap. Both stcUSD holders and Underwriters may permissionlessly realize interest by borrowing from the vault and distributing to interest receivers.

There are two functions to realize interest: <kbd>realizeInterest</kbd> and <kbd>realizeRestakerInterest</kbd>

RealizeInterest:

* Realizes vault interest (interest paid to the stcUSD holders)
* Interest is borrowed from the Vault and paid to the interest receiver

RealizeRestakerInterest:

* Realizes underwriting premium (premium paid to Underwriters)
* Interest is borrowed from the Vault and paid to the Delegation contract

For both functions, the process flow is as follows:

1. Determine available interest to realize
2. Increase reserve debt
3. Borrow interest amount from Vault
4. Transfer assets to recipient

For function signatures, parameters, and data structures, see the [Lender Contract Reference](../../developers/contracts/lender.md).
