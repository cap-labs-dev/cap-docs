# Risks

Below are the main risks associated with Cap's current design.

**Smart Contract Risk:**

* Cap operates without reliance on third-party custodians, regulatory frameworks, or manual oversight. cUSD holder protections are enforced exclusively by code-based adherence to Cap’s protocol rules. While Cap’s smart contracts have been rigorously audits, users should be aware of  risks associated with potential code vulnerabilities.

**Counterparty Risk:**

* **Shared Security Models**: Cap is built on Shared Security Marketplaces like EigenLayer. As such, it is exposed to their platform risk.
* **Reserve collateral**: If the reserve assets may depeg, users are exposed to these changes of price. Underlying reserve assets may also be frozen, seized or forfeited for illegal or sanctioned use.
* **Delegation collateral**: If the delegation assets are volatile (i.e. levered via looping), this increases the chance of liquidations as collateral value may fluctuate. Cap restricts the type of collateral to mitigate this risk.
* **Bridge and Oracle Risk**: If users wish to interact with cUSD from any chain, they will be exposed to the risk of third-party bridges that transfer cUSD from Ethereum and oracles that price cUSD.
* **Idle Asset Risk**: Idle assets may be deposited into integrated protocols such as Aave and Morpho until they are borrowed. As such, Cap is exposed to these protocol risks.

**cUSD Depeg Risk:**&#x20;

* A force majeure event may create a mismatch between cUSD and the reserve assets. However, the highly liquid and regulated nature of the reserve assets render this unlikely.

**Protocol Risk:**

* **Redemption Risk:** Redemptions are guaranteed at all times for cUSD holders. However, redemptions may be delayed if the reserve assets are fully utilized. Dynamic interest rates prevent full utilization, always ensuring liquidity to be withdrawn atomically.&#x20;
* **Liquidation Risk:** Malicious or undercollateralized Borrowers put Underwriters in liquidation risk. However, Underwriters can preemptively mitigate these risks, due to the fact that Borrowers in Cap are whitelisted regulated financial institutions.
  * **Guarantor Agreements**: Underwriters are able to sign off-chain Guarantor Agreements (GAs) with their counterparties. These agreements can specify recourse provision, loan terms, and strategy constraints. Hence, the chance of an actual liquidation event is equivalent to these regulated entities filing for bankruptcy.
  * **Quantifying risk**: It is possible to model the risk premium via credit spread. Particularly, the [return per unit of risk](https://docs.google.com/spreadsheets/d/1D04AYjvRBMCg09kSLpL8yN94tSrCpVhD8lBBT9037R4/edit?gid=0#gid=0) can be measured based on the institution's credit ratings, credit spread and default rates.&#x20;
